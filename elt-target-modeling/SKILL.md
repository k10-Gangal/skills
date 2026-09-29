---
name: elt-target-modeling
description: Build and verify data models with create_data_model so they pass automated scoring. Use when writing, creating or repairing any model in a target spec. Contains the exact rendering contract the scorer applies - timestamp to_char formats, NULL versus 0, 1/0 flags, ratio NULLIFs, surrogate keys without dbt - and a mandatory three-gate verification that catches wrong row counts and missing columns before the run ends. Use after elt-source-profiling.
---

# Building and verifying a data model

A model scores only when **every** column named in the spec matches on **every** row.
There is no partial credit. Most lost points are not hard SQL — they are render
mismatches and wrong row counts.

## What the scorer actually does

Know which mistakes are fatal and which cost nothing. Effort spent on a free property is
effort not spent on a fatal one.

### Free — do not spend a step on these

| property | why |
|---|---|
| **Extra columns** | Only spec columns are compared. Extras are ignored. Never delete a helper column to tidy up. |
| **Row order** | Both sides are re-sorted before comparison. Never `ORDER BY` in a final SELECT. |
| **String case, leading/trailing whitespace** | Compared as `a.strip().lower()`. |
| **Column-name case** | Matched case-insensitively. |
| **Numeric formatting** | `0`, `0.0`, `0.000` all coerce to the same number. Integer versus float is free. |
| **Numeric drift under 1%** | Compared with a 1% relative tolerance. Never round for cosmetics. |
| **NULL versus empty string** | Both become missing. Either is fine. |

### Fatal

| property | why |
|---|---|
| **Wrong row count** | A length mismatch fails **every column** of the model. The biggest killer by far. |
| **A missing spec column** | Fails the model. Spell it exactly as the spec spells it, typos included. |
| **NULL where a value is expected, or the reverse** | A hard fail. This is the NULL→0 rule below. |
| **Timestamp string format** | Timestamps are compared **as strings**. `2019-03-26 09:25:03` ≠ `2019-03-26T09:25:03`. |
| **A 100× scale error on a percentage** | A 1% tolerance cannot absorb it. |
| **A mixed-case table name** | Not an error — a silent zero. The whole model is invisible. |
| **An empty model** | `rows: 0` always means a broken join or an inverted filter. |

## The rendering contract

### R1 — Names and shape

Spec order, one column per line, each ending `AS <exact_spec_name>`, all lowercase and
unquoted. Never `SELECT t.*`. The SELECT list is your checklist — it must be diffable
line-for-line against the spec's column list. Specs here run past a hundred columns;
this is what keeps that tractable.

### R2 — Timestamps: obey the `format:` hint with `to_char`

A timestamp reaches the comparator as a string, and Postgres's own rendering is not
under your control. A `timestamp`-typed column is serialized at the precision of its
most precise row, so a single value carrying milliseconds silently reformats the entire
column — a column-wide coin flip you cannot see. `to_char` returns `text` and removes it.

| `format:` in the spec | emit |
|---|---|
| `YYYY-MM-DD` | `to_char(x, 'YYYY-MM-DD')` |
| `YYYY-MM-DD HH:MM:SS` | `to_char(x, 'YYYY-MM-DD HH24:MI:SS')` |
| `YYYY-MM-DD HH:MM:SS.fff` | `to_char(x, 'YYYY-MM-DD HH24:MI:SS.MS')` — exactly 3 fractional digits |
| `YYYY-MM-DD HH:MM:SS.ffffff` | `to_char(x, 'YYYY-MM-DD HH24:MI:SS.US')` — exactly 6 |
| `YYYY-MM-DDTHH:MM:SS` | `to_char(x, 'YYYY-MM-DD"T"HH24:MI:SS')` — the `T` is literal and required |
| `YYYY-MM-DD HH:MM:SS UTC` | `to_char(x, 'YYYY-MM-DD HH24:MI:SS "UTC"')` — the suffix is literal |
| `YYYY-MM-DD H[H]:MM:SS` | `to_char(x, 'YYYY-MM-DD HH24:MI:SS')` — see note |

`HH` in Postgres means 12-hour. **Always write `HH24`.** `.MS` is three digits, `.US` is
six. If the source column is text, cast inside: `to_char(x::timestamp, '…')`.
`to_char(NULL, …)` is NULL, which is what you want.

*Note on `H[H]`:* the bracket suggests the hour may be unpadded. Every sibling format in
these specs is zero-padded and Postgres pads by default, so `HH24` is the better bet.
If a model with such a column fails and nothing else explains it, `FMHH24` is the
alternative to try.

**No `format:` hint on a timestamp-looking column → pass the source text through
unchanged.** Do not cast. Source columns are usually already the right text, and a
gratuitous `::timestamp` reintroduces exactly the rendering risk you just removed.

### R3 — "Set to 0 if no X" / "replace NULL values with 0"

`coalesce(expr, 0)`. This is the most common instruction in these specs and the
highest-yield rule in this document. Gate B proves compliance: those columns must show
`n_null = 0`.

Note `count(*)` over a LEFT JOIN with no match already returns 0, but `sum(x)` returns
NULL — coalesce that one.

### R4 — "Set to 1 if …, 0 otherwise" / "Use 1 for true and 0 for false"

Emit the **integers** 1 and 0, never booleans:

```sql
CASE WHEN <condition> THEN 1 ELSE 0 END AS <name>
```

A Postgres boolean renders as `true`/`false`, the comparison falls to the string path,
and `'true' <> '1'`. Conversely, a boolean simply *passed through* from source with no
such instruction stays as it is — case is matched case-insensitively.

### R5 — Derived ratios: `NULLIF`, not `coalesce`

```sql
<numerator>::numeric / NULLIF(<denominator>::numeric, 0) AS <name>
```

A zero denominator yields **NULL**, not 0, unless the description explicitly says to set
it to 0. Cast text columns before dividing — integer division silently truncates. Any
average or ratio goes through `::numeric`.

### R6 — Percentages: pick a scale and commit

A 100× error always fails, and an extra column cannot hedge a named one. Decide from the
wording:

- the description says **"percentage"** → emit **0–100** (`100.0 * n / nullif(d,0)`);
  *per cent* means per hundred
- it says **rate / ratio / proportion / share** → emit **0–1**

Then confirm with Gate B: that column's `max_text` tells you which scale you produced.
This is a judgement call the spec does not always settle. Make it, note it, move on.

### R7 — "with ties broken by …"

The tiebreaker goes **inside the window's `ORDER BY`**, after the primary sort. This is a
value-level requirement, quite separate from output row order, which is free:

```sql
row_number() over (partition by <key> order by <primary>, <tiebreak>) AS <name>
```

Getting it wrong changes the numbers in the column, not the order of the rows.

### R8 — Surrogate keys without dbt

When a description says to use the dbt utils package to generate a surrogate key, there
is no dbt here. `dbt_utils.generate_surrogate_key` is plain md5 over the listed fields
joined by `-`, and it reproduces directly:

```sql
md5( <f1_text> || '-' || <f2_text> || '-' || <f3_text> )
```

Three rules:

1. **Order the fields exactly as the description names them.** A permutation produces a
   completely different hash.
2. **Hash the text you would emit for that field.** md5 is over text, so any cast changes
   the result. If the field is a timestamp with a `format:` hint, hash its `to_char`
   rendering, not a re-cast value.
3. **Compute the key inside the CTE that owns the entity, then LEFT JOIN it in.** A row
   with no matching entity then carries NULL naturally. Only where a component is
   genuinely null *within* its own entity does dbt_utils' placeholder apply:
   `coalesce(<field_text>, '_dbt_utils_surrogate_key_null_')`.

### R9 — JSON

Nested values from `.jsonl`-derived tables are JSON text. Use `col::json ->> 'field'`.
`->>` returns text; `->` returns JSON and renders with quotes, which fails the compare.

### R10 — Fallbacks

"use X; if X does not exist, use Y" → `coalesce(nullif(x,''), y)`. Guard against the
empty string as well as NULL — the loader stores both.

## Build with staged CTEs

Do not write one flat query per model. Stage it:

```sql
WITH base      AS ( … the driving entity, one row per grain key … ),
     child_agg AS ( … pre-aggregate each 1:N child to the grain key … ),
     enriched  AS ( … join base to each *_agg, all LEFT JOINs … )
SELECT  … one spec column per line, spec order, AS exact_name …
FROM enriched
```

Pre-aggregating every child to the grain key **before** joining is what prevents
fan-out, the commonest cause of a wrong row count. It also makes each `coalesce(…, 0)`
obvious, because you can see which LEFT JOIN is allowed to miss. One statement per call
either way, so a CTE chain costs no more than a flat query and is far easier to correct
in place.

## Build the widest sibling first

When several models in a database share most of their column names — several reports at
different grains, an overview and an analysis over the same entity — build the **widest**
one, verify it through all three gates, then derive the siblings by changing the grain in
the GROUP BY and deleting SELECT lines. A verified sibling then needs one create and
three gates, not a fresh design. This is where a very wide database gets cheap.

## Create, then verify. Do not dry-run.

`create_data_model(database, model_name, select_sql, replace=True)` returns the row
count, the full column list with types, and sample rows — strictly more than a
`run_read_query` of the same statement, for the same one call — and `replace=True` makes
it freely repeatable. So skip the dry run and create directly. (The tool docstrings
suggest testing first; for this scoring setup that is a wasted step.)

Two constraints on `select_sql`: it must be a bare `SELECT` or `WITH`, never
`CREATE TABLE … AS`; and one statement, no semicolon batching.

**One hazard.** A failed `create_data_model` has already dropped the previous version and
leaves **no table at all**. The final action on every model must be a *successful*
create. Never end a run with a model in that state.

## The three gates — run all three after every create

All three run through the read-only server's `run_read_query`. **The read-only role can
see the models you create** — SELECT is granted on them by default, precisely so you can
do this. Nothing in the tool descriptions says so, so it is easy to miss: you have
arbitrary SQL over your own output. Use it.

### Gate A — grain. One call.

```sql
SELECT (SELECT count(*)                    FROM <db>.<model>)                    AS model_rows,
       (SELECT count(DISTINCT (<k1>,<k2>)) FROM <db>.<model>)                    AS distinct_grain,
       (SELECT count(*)                    FROM <db>.<model> WHERE <k1> IS NULL) AS null_key,
       (SELECT count(DISTINCT (<k1>,<k2>)) FROM <db>.<driving_source>)           AS source_grain
```

Pass iff `model_rows = distinct_grain = source_grain` and `null_key = 0`.

| symptom | cause | fix |
|---|---|---|
| `model_rows > distinct_grain` | a join fanned out | pre-aggregate the N side in a CTE |
| `model_rows > source_grain`, `= distinct_grain` | missed dedupe | `row_number() … = 1` |
| `model_rows < source_grain` | an INNER JOIN dropped rows, or WHERE over-filtered | drive from the base table with LEFT JOINs |
| `null_key > 0` | driving from the wrong side | swap the driving table |
| `model_rows = 0` | broken join or inverted filter | always a bug |

Run this **before** any column work. A wrong row count fails every column, so column
fixes on a wrong-grain table are wasted steps.

### Gate B — every column, one call, any width.

```sql
SELECT e.k                                         AS column_name,
       count(*)                                    AS n_rows,
       count(*) FILTER (WHERE e.v = 'null'::jsonb) AS n_null,
       count(DISTINCT e.v)                         AS n_distinct,
       min(e.v #>> '{}')                           AS min_text,
       max(e.v #>> '{}')                           AS max_text
FROM <db>.<model> t, LATERAL jsonb_each(to_jsonb(t)) AS e(k, v)
GROUP BY e.k
ORDER BY e.k
```

One row per column, so a 118-column model is still one call. Check four things:

1. **Presence** — the `column_name` set must contain every spec column name. This is your
   inventory. Do not re-read your own SQL to count columns.
2. **NULL→0** — every "Set to 0 if" / "replace NULL with 0" column must show `n_null = 0`.
3. **Format** — for each `format:` column, `min_text` and `max_text` are literal rendered
   values. Check the separator (space versus `T`) and the count of fractional digits,
   character by character.
4. **Degenerate columns** — `n_null = n_rows` is an all-NULL column, almost always a
   typo'd source reference or a join that never matches. `n_distinct = 1` on something
   that should vary is the same signal.

**`min_text` and `max_text` sort as text, not as numbers.** On a numeric column they will
read like `min 1, max 9` across thousands of rows. That is correct and expected — they
are there to show you the *rendering*, not the range. Do not read them as a numeric
min/max, and do not "fix" a column because of them.

### Gate C — read the values. One call.

If Gate A reported a small `model_rows` (roughly 25 or fewer):

```sql
SELECT * FROM <db>.<model>
```

Otherwise re-read only the columns Gate B flagged:

```sql
SELECT <flagged columns> FROM <db>.<model> LIMIT 20
```

Gate C catches what aggregates cannot: a tie-break that produced the wrong sequence
number, a coalesce on the wrong branch, a ratio inverted.

### The loop

```
create_data_model(replace=True)  →  Gate A  →  Gate B  →  Gate C
    any gate fails  → fix the SELECT → create again
    all three pass  → next model, and do not revisit
```

## Finishing

- Where the spec is ambiguous, pick the reading that satisfies the grain sentence and the
  most column descriptions, state the assumption in one line, and continue. Ask instead
  only if your task prompt says someone is available to answer — if it is silent on that,
  assume nobody is, and decide rather than stall.
- Before stopping, call `list_data_models(database)` once and confirm every model in the
  spec exists. A model that was never created can never score; an imperfect one still
  might.
- If you are running short on budget with models remaining, drop to the minimum loop —
  create + Gate A + Gate B — and skip Gate C. Existing beats polished.

## Rules that never change

- Lowercase, unquoted identifiers. Models go in the **same** schema as the source tables.
- One SQL statement per call, no trailing semicolon.
- Never `ORDER BY` in a final SELECT — row order is free and it wastes work.
- Never delete an extra column to tidy up; extras cost nothing and the deletion might
  take a real one with it.
- Avoid the letter sequence `copy` at the start of an alias or CTE name; the SQL guard
  rejects it as a word match.
- Read the whole error string. It usually contains the fix, including the exact column
  name you meant.
