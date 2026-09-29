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

## The agent prompt

[`PROMPT.md`](PROMPT.md) is the system prompt the two skills were written for. Sections:
introduction, skills, workflow, mandates, examples, target specification.

It is not required — the skills stand alone and work under any prompt that attaches them.
It is here because the split between the two matters: **per-run facts live in the prompt,
stable rules live in the skills.** The prompt owns the budget, the database, the target
spec, and whether a human is available to answer a question. The skills own the rendering
contract and the verification gates, identical on every run.

The prompt branches on whether a target spec was supplied:

| | spec pasted | spec absent |
|---|---|---|
| Step 2 | build it, ask nothing | profile, propose a plan, ask for a spec, agree columns, *then* build |
| needs a human | no | **yes — it will stop and wait** |

Two details in it are load-bearing and easy to undo by accident. The skills directive sits
*before* the paragraph naming the tools, because an agent acts on the earliest instruction
it can execute — lead with the tools and you have handed it an opening move that isn't
loading a skill. And "every **tool call** is pre-approved" is scoped to tool calls on
purpose; unscoped, it cancels the branch above by telling the agent never to ask anything.
