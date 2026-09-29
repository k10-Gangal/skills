---
name: elt-source-profiling
description: Profile source tables in a Postgres schema before writing transformation SQL. Use at the start of every database, before building any data model, when a task supplies a target spec and asks for data models. Establishes each model's grain and expected row count, mines column meanings from describe_table, picks join keys that will not fan out, and holds exploration inside a fixed budget.
---

# Source profiling

Answer three questions about this database before you write a line of SQL. Nothing else
in exploration earns anything.

1. **How many rows should each target model have?** A wrong row count fails every column
   of that model. It is the most expensive thing to get wrong and the cheapest to check.
2. **Which source table drives each model, and what is its key?**
3. **What do the source columns actually mean?** Names like `value`, `totals`, `flag_3`
   are unguessable. Their descriptions are not.

## Where the target spec is

**In your task prompt, and nowhere else.** No tool returns it. Do not spend a call
looking for it.

`describe_table` gives you descriptions of *source* columns — what the inputs mean. It
never tells you what an output column should be called or what it should contain. Those
come only from the spec in your prompt.

## The exploration budget

Hard caps. Exceeding them costs more than the information is worth, because every tool
result stays in your context and is re-sent on every later turn. Cost, not step count,
is what actually runs out.

| tool | cap per database | when |
|---|---|---|
| `list_tables` | 1 | always, first call |
| `describe_table` | 10 on the first pass | only tables the spec plausibly needs |
| `profile_column` | 4 | join keys, and columns whose encoding the samples hide |
| `preview_table` | **0 — never call it** | `describe_table` already returns sample rows |

Go past the `describe_table` cap only when a specific spec column cannot be located in
any table you have already described. That is a good reason; curiosity is not.

## Step 1 — `list_tables(database)`, and read the row counts

One call returns every table with its live row count. Those counts are a free grain
oracle: a target model's row count usually **equals the row count of one source table**,
or the distinct count of one column in it. When you compute an expected row count in
step 3, check it against this list. A match is strong confirmation. A mismatch means you
picked the wrong driving table or missed a dedupe.

## Step 2 — fix the grain, per model, in writing

State the grain explicitly before designing anything. Skipping this is the single most
common way a model ends up quietly wrong.

**Primary source: the model's `description` sentence in the spec.**

| sentence | grain |
|---|---|
| "Each record represents an account." | one row per account id |
| "Each record represents the daily performance of an ad." | one row per (date, ad id) |
| "Each record represents a distinct session…" | one row per session key |

**Fallback when there is no such sentence.** Some specs open descriptively instead —
"Detailed view of all surveys created…", "A directory is an address book…". Then the
grain key is the `<entity>_id` whose entity name matches the model name, and it is
almost always the first column the spec lists.

**Modifiers that change the grain, each with a required action:**

| phrase in the spec | what it means | what you must do |
|---|---|---|
| "Use the most recent record if multiple exist for the same X" | the source is **not** unique on X | dedupe: `row_number() over (partition by X order by <recency> desc) = 1` |
| "…enriched with metrics about their associated Y" | Y is a 1:N child | pre-aggregate Y in a CTE, then join 1:1 — never join raw |
| "daily performance of Z" | compound grain | `(date, Z)`; the date column usually carries a `format:` hint |

## Step 3 — confirm the expected row count with one query

```sql
SELECT count(*)                              AS src_rows,
       count(DISTINCT <key>)                 AS distinct_keys,
       count(*) FILTER (WHERE <key> IS NULL) AS null_keys
FROM <db>.<driving_table>
```

- `src_rows = distinct_keys` → that number is your expected model row count.
- `src_rows > distinct_keys` → the source is not unique on this key. Either the spec told
  you to dedupe, or this is the wrong driving table.
- `null_keys > 0` → those rows cannot appear under that key. Find out why before building.

**Write the number down.** Gate A in `elt-target-modeling` will hold you to it.

### `_fivetran_deleted`

Some source tables carry it. Default: **do not filter on it**. If and only if your
expected row count disagrees with the model's, re-run the count with
`WHERE _fivetran_deleted IS NOT TRUE` and see which number is right. `_fivetran_synced`
and `_fivetran_id` are load metadata and never belong in a data model unless the spec
names one. There are no `_airbyte_*` columns in this warehouse.

## Step 4 — `describe_table` on the tables you will actually use

Read three things from each result and nothing else.

1. **`description` per column.** The only place source semantics exist. A source column
   whose description mentions the same noun as a target column's description is your
   match.
2. **`type`.** Types were inferred at load time and a column only narrowed to a number or
   a timestamp if *every* non-empty value parsed. Assume `text` until told otherwise.
   Cast at the point of use — `(t.amount)::numeric` — not in a wrapper CTE you then
   forget about. `''` is not NULL: use `nullif(x,'')::numeric` before you `sum` or `avg`.
3. **The sample rows.** They show the real encoding: `'Y'`/`'N'` versus `true`/`false`,
   `2019-10-02 16:02:54` versus `2019-10-02T16:02:54.000Z`, JSON kept as text. This is
   what decides your casts.

**Look at a timestamp sample before deciding to reformat it.** If the source text already
matches the target's `format:` exactly, pass it straight through. Casting to `timestamp`
and back is how fractional seconds get lost.

### Nested / JSON columns

Tables loaded from `.jsonl` keep nested values as JSON **text**. Extract with
`col::json ->> 'field'` — `->>` returns text. `->` returns JSON and renders with
surrounding quotes, which fails a string comparison.

## Step 5 — `profile_column` on join keys only

Four calls maximum. Use it for facts the samples cannot give you.

| reading | meaning |
|---|---|
| `distinct = total_rows` | a key; safe to join on, no fan-out |
| `distinct < total_rows` | joining on it **will** fan rows out; pre-aggregate first |
| `nulls > 0` | an INNER JOIN silently drops these rows; `count`/`avg` skip them |
| `most_common` | the real encoding of a flag or category |

A key checked here costs one call. The same problem found later — from a wrong row
count — costs a full rebuild.

## Hand-off

Before switching to `elt-target-modeling`, have this written down for each model:

- grain, in words
- driving source table, and its key
- expected row count, as a number
- for each target column, which source table and column feeds it, or "derived"
- **any column you could not place** — decide now, do not leave it blank

An unplaceable column is the main legitimate reason to spend a call past the caps. A
missing column fails the whole model; an extra one costs nothing.

## Rules that never change

- All identifiers lowercase and unquoted. A mixed-case table name is not an error — it
  is **invisible** to scoring.
- Always schema-qualify: `<db>.<table>`.
- Your data models go in the **same** schema as the source tables, the one named after
  the database. Do not create a separate schema for outputs.
- Never modify a source table. Postgres will refuse anyway.
- One SQL statement per call, no semicolon batching.
- Results are capped at 200 rows and `max_rows` can only lower that. `truncated: true`
  means rows were dropped from the *response*; it says nothing about whether your query
  is right. Establish facts with aggregates; use rows only to see shape.
- Avoid the letter sequence `copy` at the start of any alias or CTE name — the read guard
  matches it as a word and will reject the statement.
- A tool error is a complete instruction. `column "amont" does not exist — Perhaps you
  meant to reference the column "o.amount".` *is* the fix. Apply it; do not re-derive the
  query from scratch.
