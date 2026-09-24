---
name: create-dataset-index
description: >
  Build a datasets index for a project — a README index plus per-table
  profiles for the warehouse tables the project's SQL work touches, in the
  shape of a reviewed data-warehouse documentation tree. Two entry modes:
  mine a repository full of SQL files, or start from tables and queries the
  user names in chat. Knowledge gaps (column types, meanings, partition
  columns) close through an iterative probe loop where the user runs
  copy-pasteable queries in their analytics workspace (e.g. Mode
  Analytics) and pastes results back. Use when asked to create a dataset
  index, document warehouse tables, build table profiles, or set up this
  documentation tree for a project.
license: MIT
metadata:
  version: "0.1.0"
---

# Create Dataset Index

Build a `datasets/` documentation tree in the user's project: a `README.md`
that indexes every warehouse table the team touches, plus one profile per
table under `tables/<project-slug>/<dataset>/<table>.md`. This skill encodes a
working shape that has survived real use; follow it rather than inventing a
new one, and adapt only what the user's warehouse genuinely differs on.

The warehouse is assumed to be **BigQuery** and the probe templates are written
in its dialect. If the user's warehouse differs, adapt the SQL per template and
say so in each profile's provenance lines.

## The hard rule

**The agent never connects to the warehouse. Not once, not with a shortcut, not
"just this probe".** Agent-to-warehouse access is prohibited even when the
user's machine holds working credentials — a client library, a service-account
file, an authenticated session, all of it. None of that changes anything: every
probe query is written by the agent, run by the user in their analytics
workspace (Mode Analytics or similar), and pasted back into the chat. The
back-and-forth is not a fallback for users without access; it is the only
route the skill has.

This rule outranks every other instruction in this skill. If any step, probe,
or convenience ever seems to offer a direct connection, stop and reread this
section.

## Why the index exists

The index is not documentation for its own sake — it is what lets an agent
work with the user on queries:

1. **Drafting a new query from natural language.** A request like "monthly
   settled transactions by province" becomes a working query only if the agent
   knows which tables exist, their full paths (copyable from every profile),
   which columns mean what, and how the tables join. Key fields and
   Relationships are what turn a sentence into SQL without guessing.
2. **Refining an existing query.** The Partitioning section tells the agent
   which filter keeps the scan cheap, First seen in / Used in show how the
   team already queries these tables, and verified column types prevent
   silently wrong casts.

Everything the loop probes is in service of those two jobs: the better the
index, the fewer round-trips between the user's intent and a correct, cheap
query. When a hole would not change how queries get written, it is a
low-rank hole — the ranking in Step 4 follows from this purpose.

## The shape you are producing

```
<project>/
└── datasets/              # the index root
    ├── README.md          # the index (see below)
    └── tables/
        └── <project-slug>/
            └── <dataset>/
                └── <table>.md    # one profile per table
```

The **README** carries: a one-paragraph opening (what this index is, how
profiles are organized), the schema-discovery how-to with the probe templates
(copy-pasteable — this is where the probe loop's queries live), the
project-ID mapping (slug → real project), the table quick-reference (Table |
Type | Purpose | Profile), an unreviewed-datasets note, and the table profile
template.

Each **profile** is:

```markdown
# `<dataset>.<table>`

- **Full path**: `<project_id>.<dataset>.<table>`
- **Type**: TABLE / VIEW / WILDCARD VIEW / MATERIALIZED VIEW

## Purpose
...
## Business purpose
...
## Key fields
- `column` (TYPE) — description
...
## First seen in
...
## Used in
...
## Partitioning
- **Probe** (get_table or INFORMATION_SCHEMA, <date>): what the object reports.
- (dated dry-run or filtered-query evidence where it exists)
## Relationships
- Joined with `<other_table>` on `<key>`
```

House rules for every profile, non-negotiable:

- **Identity first**: the title is `dataset.table`; the Full path bullet carries
  the fully-qualified name, because that is what a reader copies to query.
- **Type is the physical kind only** — TABLE, VIEW, WILDCARD VIEW,
  MATERIALIZED VIEW — verified against metadata, never guessed from prose.
  Passthrough provenance ("VIEW, passthrough to `...`") rides in the Type line.
- **Key fields** hold the complete column list where the schema is verified;
  columns without a known meaning stay bare (`column` (TYPE)), never invented.
  Where only a count is known, a closing sentence states it.
- **Partitioning** is its own section, focused on the fact (which column, what
  granularity, what clustering), with the verification evidence inside it —
  the section is named for the fact, not for the act of verifying it.
- **Facts carry dates and methods**. "12.5M rows" is unfinished; "12.5M rows
  (COUNT probe, 2026-01-15)" is a fact. Every probe answer lands with its date.
- **Purpose and Business purpose answer different questions**, not the same
  one twice: Purpose is one line on what the table contains (content and
  grain, the technical answer); Business purpose is why it earns its place in
  the team's workflows, the consumer and the payoff, the "so that" sentence.
  Key fields sit after both, because by then the reader knows what the table
  is and why the columns matter. Business motivation before technical fact;
  write both for a junior data scientist joining the team.
- **No em-dashes**, table nulls are `n/a`, and nothing is invented: a profile
  grows only from SQL evidence, probe results, or a user statement. A hole
  stays a hole until filled, visibly.

## Mode selection

Start by asking one question: **where does the user's SQL live?**

- **Mode 1 (default)**: the user has a repository full of SQL files
  (download scripts, analysis queries, pipelines). Mine it. Ask for the
  directory (or accept the repo root) and go to Step 1.
- **Mode 2**: no such repo. Ask the user, in chat, for either (a) the paths of
  the tables they mostly work with, or (b) a few representative queries they
  run often. Either is enough to seed the index; go to Step 1 with those.

Establish nothing about access, because one rule covers it: **the agent never
connects to the warehouse, ever.** Even when the user's machine holds working
credentials, direct agent-to-warehouse access is prohibited. The only route
for knowledge is the back-and-forth — the agent writes copy-pasteable probe
queries, the user runs them in their analytics workspace (Mode Analytics or
similar) and pastes results back.

## Step 1: Establish the mapping and the skeleton

1. Confirm the index root (default: `datasets/` at the repo root) and create it.
2. Collect the project slugs: every fully-qualified table reference maps a
   slug (a short name like `prod`) to a real project ID. The README's
   Project ID Mapping section is the single place that resolves slugs, and
   profile paths encode them.
3. Write the README skeleton (all sections, empty index).

## Step 2: Mine the SQL

Read every SQL file the user points at (in Mode 2: the pasted queries) and
extract, per table referenced in `FROM` / `JOIN` clauses:

- **Provenance**: which file first references it, and every file that uses it.
  These become First seen in / Used in — the cheapest, most durable facts.
- **Joins**: join keys and join types become the Relationships section.
- **Partition evidence**: any query that filters a large table on one column
  (`WHERE created_at >= ...`) is evidence for a partition candidate, with the
  file and line as provenance. Record it as evidence, not as fact, until
  metadata confirms it.
- **Column usage**: selected, filtered, and aggregated columns are candidates
  for Key fields, with the query context as the first draft of a meaning.
- **Semantic hints**: CTE names, table aliases, and file purposes describe
  what the table is for — draft Purpose and Business purpose from them, and
  mark every draft as a draft.

Write the profile skeletons now: identity list (ask the user or infer the
project slug; Type stays a hole until probed), the mined sections, and holes
left visibly open. Build the README's quick-reference as you go, one row per
profile, linking to it. Nothing is invented; skeletons are honest about gaps.

## Step 3: The knowledge ledger

The loop's ledger lives in the **conversation itself**, not in a file: the
running account of what is known per table, what is still a hole, the ranked
probe queue, and what has been asked. Announce it in one or two lines before
every probe round, so both sides always see where the build stands.

The durable record is the profiles on disk. Every answered probe lands in its
profile in the same round, with its source and date, so the tree itself
accumulates the knowledge; holes are just sections that stay thin or absent.
When the user pauses or the session ends, nothing is lost: the profiles hold
every known fact (each dated), and the visible gaps are the remaining holes.

Resuming is therefore a read, not a restore: a new session re-inventories the
built tree, re-derives the ranked holes from what the profiles are missing,
and continues the back-and-forth without re-asking anything the profiles
already carry. A hole the user declined to probe is recorded where it belongs,
in that profile ("not investigated: <reason>"), so the judgment survives
sessions too.

## Step 4: The probe loop

This is the heart of the skill, and it is a conversation, not a script.

**Rank the holes.** Across all tables, order them by importance:

1. **Partition column** — the cost-critical fact. A scan without partition
   pruning on a large table is the difference between a query that costs
   cents and one that costs tens of dollars. Every large table's profile
   needs this.
2. **Object type** — is it a TABLE, a VIEW, a MATERIALIZED VIEW? The answer
   changes everything downstream: views report no partitioning of their own,
   and a wrong Type assumption misleads every later query.
3. **Column list with types** — the schema that Key fields carries.
4. **Column meanings** — drafted from SQL evidence, confirmed by the user.
5. **Relationships** beyond what joins in the SQL show.
6. **Example values and row counts** — grounding, when a meaning is doubtful.

**Ask in small batches.** One to three probe queries per round, always for
the highest-ranked hole, formatted to copy-paste. Announce the loop's state
before the probes: what is known, what each probe will close.

**Every probe is copy-pasteable SQL** the user runs in their analytics
workspace and pastes back. Use the templates below, fill in the placeholders,
and never ask the user to modify a probe beyond filling a table name.

**Propose meanings before probing for them.** SQL evidence often already
suggests what a column means; draft the meaning, show the evidence, and ask
the user to confirm or correct. A confirmation is cheaper than a sample-row
probe and teaches the index the team's vocabulary.

**The user can pause at any time** by moving to another session. Say so once
when the loop starts: the profiles on disk are the durable state, so a future
session resumes by re-inventorying the tree and re-deriving the ranked holes
from what the profiles visibly lack. Nothing the user already answered is ever
re-asked; every probe answer carries its date inside a profile.

**Close the loop when the user stops adding value**, not when every hole is
filled: some holes (a deprecated table's internals, an archive nobody queries)
are cheaper to mark "not investigated" than to probe. Record that judgment in
the profile itself, with its reason.

## Probe templates

### Schema probe (columns and types)

Closes hole 3: the column list with types and nullability, plus any
warehouse-maintained descriptions that feed hole 4's drafts.

```sql
SELECT
  column_table.column_name,
  column_table.data_type,
  column_table.is_nullable,
  field_table.description
FROM
  `<project>.<dataset>.INFORMATION_SCHEMA.COLUMNS` AS column_table
LEFT JOIN
  `<project>.<dataset>.INFORMATION_SCHEMA.COLUMN_FIELD_PATHS` AS field_table
ON
  column_table.table_catalog = field_table.table_catalog
  AND column_table.table_schema = field_table.table_schema
  AND column_table.table_name = field_table.table_name
  AND column_table.column_name = field_table.column_name
  AND field_table.field_path = column_table.column_name
WHERE
  column_table.table_name = '<table_name>'
ORDER BY
  column_table.ordinal_position;
```

The `field_path` condition keeps one row per top-level column: without it, a
STRUCT column repeats once per nested field. The nested fields and their types
still show inside the column's `data_type`. If the warehouse documents nested
fields, drop the condition to see their descriptions.

### Table-type and DDL probe (object type, declared partitioning, view lineage)

Closes hole 2, and gives hole 1 its first evidence. It is the cheapest probe
in the set: batch it across every table of a dataset in one query with `IN`.

```sql
SELECT
  table_list.table_name,
  table_list.table_type,
  table_list.ddl
FROM
  `<project>.<dataset>.INFORMATION_SCHEMA.TABLES` AS table_list
WHERE
  table_list.table_name IN ('<table_a>', '<table_b>');
```

Read the `ddl` for three facts:

- **For a table**: the `PARTITION BY` clause names the partition column and
  its granularity (`DATE(event_time)`, or `_PARTITIONDATE` for
  ingestion-time partitioning), and `CLUSTER BY` names the clustering
  columns. Record them as declared, then confirm with the dry run.
- **For a view**: the DDL has no partitioning of its own, but its `FROM`
  clause names the backing table. That table is where the partitioning
  lives, so it is the next thing to probe.
- `table_type` reports `BASE TABLE` for a plain table; the profile's Type
  line records it as TABLE.

Metadata access is granted per dataset: if `INFORMATION_SCHEMA` returns
"Access Denied" for a dataset (common for a view's backing dataset), skip
straight to the dry-run probe below for its tables.

### Partition probe (dry run)

Closes hole 1 empirically: it confirms what the DDL declares, and it is the
only route when the DDL is unreadable — the common case in organizations
where most tables are views: the view reports no partitioning and its
backing dataset denies metadata access, yet the backing table prunes fine.

The workspace typically shows neither bytes processed nor runtime, so the
empirical probe goes through the org's dry-run function (e.g.
`functions.dry_run` in a shared `functions` dataset): it asks BigQuery for a
query's scan estimate without executing the query, callable as plain SQL from
any surface. Confirm the function's exact name once when the loop starts;
if the workspace provides none, fall back to a runtime comparison as a weak
signal.

Ask the user to run both calls and paste both results:

```sql
-- Probe A (unfiltered):
SELECT `functions.dry_run`(
"""
SELECT *
FROM `<project>.<dataset>.<table>`
"""
)

-- Probe B (filtered on the suspected partition column):
SELECT `functions.dry_run`(
"""
SELECT *
FROM `<project>.<dataset>.<table>`
WHERE <column> >= <recent_timestamp_or_date>
"""
)
```

A large drop in reported bytes between B and A confirms the column prunes; no
drop means it is not a partition column, or the predicate is not the pruning
one. Because a dry run executes against the view's definition, this works
through views too — it measures the behavior that matters, not the wrapper.
Run both in the same session so the comparison is fair, and record both
numbers with their date in the profile's Partitioning section.

### Meaning probe (sample rows)

Closes hole 6's grounding when names and user confirmation are not enough.

```sql
SELECT * FROM `<project>.<dataset>.<table>`
WHERE <partition_column> >= <recent_date>
LIMIT 5;
```

Always carry the partition filter once it is known, so the meaning probes
stay cheap. Ask the user to describe anything the columns' names alone do not
explain; their domain knowledge is a source the profile should cite.

## Step 5: The README

When the probe loop winds down, finish the README. Read and adapt the bundled
template at `references/readme-template.md`: it carries the full skeleton —
the opening, the schema-discovery and partition-probe how-tos (the same
templates the loop used, so future readers can re-verify), the project
mapping, the quick-reference table, the unreviewed-tables note, and the
profile template. Fill every placeholder from what the build learned, and
delete the optional sections the project does not need. The index is
maintainable only if re-probing is documented, not just performed once.

## Working rules

- **Write profiles as you learn, not at the end.** Every answered probe
  updates the profile files in the same round; the conversation only ever
  shows the diff-sized delta.
- **Never invent.** A column meaning without evidence is a hole, marked as
  one. A guessed partition column is worse than no partition column, because
  a wrong filter is a silent full scan.
- **Never connect to the warehouse — no exception.** Even when the user's
  machine holds working credentials, agent-to-warehouse access is prohibited:
  every probe is run by the user through their analytics workspace and pasted
  back. The back-and-forth is the only route, not a fallback.
- **Never commit.** The user reviews the tree with `git status` / `git diff`
  and commits when satisfied.
- **Batch honestly.** Three probes per round is a courtesy to the user's
  time, not a cap on curiosity; the conversational ledger keeps the queue so nothing is
  asked twice or forgotten.
- When the index is done, the measure is: N tables profiled, N probes run,
  M holes open with their reasons, and the README's quick-reference resolving
  every row to a real profile.