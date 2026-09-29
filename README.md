# Skills

Agent skills, one directory each, in the standard `SKILL.md` format — YAML frontmatter
(`name`, `description`) followed by a markdown body. Skills are progressively disclosed:
only the `description` sits in the agent's context until it invokes one, so each skill
stands alone and states its own preconditions.

## ELT / data modeling

A pair for agents that build SQL data models against a spec and are scored by an
automated comparator. They split on **what I can learn** versus **what I must emit**.

| skill | fires | answers |
|---|---|---|
| [`elt-source-profiling`](elt-source-profiling/SKILL.md) | once per database, before any SQL | How many rows should this model have, and what do the source columns actually mean? |
| [`elt-target-modeling`](elt-target-modeling/SKILL.md) | once per model, and on every repair | How do I render each value so the comparator accepts it — and how do I prove I did? |

Attach both. Neither assumes the other has loaded, so some overlap between them is
deliberate.

**What they contain.** Generic workflow only: how to fix a model's grain before writing
SQL, how to hold exploration inside a budget, the rendering rules a string-based
comparator actually enforces (timestamp formats, NULL versus 0, integer flags, ratio
guards), and a three-gate verification loop that reads a model back after creating it —
row count, then every column in a single query, then values. No per-database knowledge
and no benchmark answers.

**What they assume.** Postgres, and two MCP servers: one read-only over the source
schema, one that creates tables. The read-only role must have SELECT on the tables the
write role creates — that grant is what makes verification possible, and the skills say
so explicitly because no tool description implies it.

The gate-B query uses `jsonb_each` over `to_jsonb(row)`, so it needs PostgreSQL 9.4+;
it was developed against 16.
