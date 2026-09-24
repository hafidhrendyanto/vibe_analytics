---
name: pair-analytics
description: >
  Pair with the user on warehouse analytics through the project's datasets
  index: turn a natural-language request into a correct, cheap BigQuery query,
  write it to a file, and hand it to the user to run in their analytics
  workspace (Mode Analytics) — the agent never connects to the warehouse.
  Small results come back pasted for debugging and stepping stones; large or
  notebook-bound results come back as a downloaded CSV in the repo, followed
  by a proposed notebook (marimo preferred, Jupyter fine) that shows the
  insight. Use when asked to write or refine warehouse queries, pull data for
  analysis, explore indexed tables, or turn a data question into a notebook.
license: MIT
metadata:
  version: "0.1.0"
---

# Pair Analytics

Pair with the user on warehouse analytics: turn what they ask for in natural
language into a correct, cheap BigQuery query, get the result back through
them, and turn the result into an insight they can react to.

This skill assumes the project carries a **datasets index** — a README plus
per-table profiles built by the `create-dataset-index` skill. The index is
the knowledge source: full paths, column meanings, partition columns,
relationships. If the project has no index, propose building one first with
`create-dataset-index`; working against an undocumented warehouse is exactly
the guesswork this pairing exists to remove.

## The hard rule

**The agent never connects to the warehouse. Not once, not with a shortcut, not
"just this query".** Agent-to-warehouse access is prohibited even when the
user's machine holds working credentials — a client library, a service-account
file, an authenticated session, all of it. None of that changes anything: the
agent writes the query to a file, the user runs it in their analytics
workspace (Mode Analytics or similar), and the result comes back through the
user — pasted into the chat or downloaded into the repo. The user's run is
the only execution the skill has.

This rule outranks every other instruction in this skill. If any step seems to
offer a direct connection, stop and reread this section.

## What the index gives each query

Before writing any query, read the profiles of the tables the request
touches. The index answers, without guessing:

- **Full path** — the copyable `project.dataset.table` for every FROM clause.
- **Key fields** — the column names, types, and meanings that turn the
  request's words into real column names.
- **Partitioning** — the filter that keeps the scan cheap. A query on a large
  table without its partition filter is the difference between a read that
  costs cents and one that costs tens of dollars; never emit one without it
  once the column is known.
- **Relationships** — the join keys the team already uses.
- **First seen in / Used in** — how the team queries these tables today; a
  refinement should start from the cheapest correct existing shape.

If a needed fact is missing from the index (a column the profiles do not
carry, a partition column not yet verified), do not guess: propose the probe
round from `create-dataset-index` to fill the hole, or ask the user directly.
The gap becomes a hole in the index, not a silent assumption in the query.

## The loop

1. **Restate the request as a query plan** — the tables, the filters, the
   grain, the expected result shape. One or two lines, so the user corrects
   the aim before any SQL is written.
2. **Write the query to a file**, one file per query, named after what it
   answers. The default home is `datasets/analytics/`: pick the child
   directory inside it that fits the analysis best (an existing one such as
   `datasets/analytics/retention/`, or a new child named after the
   work when none fits). Write it in the bundled SQL patterns — read
   `references/sql-patterns.md` before the first query of a session. The user
   copies the query from the file into Mode Analytics and runs it there.
3. **The result comes back through one of two channels** — Mode's own
   result actions decide which:

   - **Copy result** (clipboard, pasted into chat): for small results, and
     for anything that is debugging or a stepping stone toward a more final
     query. Iterate fast; nothing lands on disk yet.
   - **Download CSV** (moved into the repo): when the result is large, or it
     is going to be used in a notebook. Ask the user where to put it if the
     project has no convention; a predictable home inside the same
     `datasets/analytics/` child keeps the data next to the query and the
     notebook that reads it.

   State the recommendation, then ask — the user knows whether the result is
   a stepping stone or a keeper.

4. **Propose the notebook — or the quick answer.** When a CSV lands, propose
   a notebook that reads it and shows the insight: the user can see the data
   and propose new changes from there. But when a quick answer is needed, or
   the user asks for one, answer in chat instead — a short TL;DR with a
   markdown table (or a small chart sketch) beats a notebook for a question
   that only wants answering; the notebook can follow once the analysis
   grows. Ask two questions before creating one:

   - **Marimo or Jupyter?** Marimo is preferred, driven live with the
     `marimo-pair` skill when it is available; Jupyter is fine when the
     user's stack prefers it.
   - **Polars or pandas?** Polars is preferred; pandas is fine when the user
     asks for it.

   The notebook reads the downloaded CSV — never queries the warehouse, never
   calls the network. It is the lens on the result, not a second execution
   path. And it logs the query behind each CSV it reads: the SQL itself or a
   pointer to its query file, right where the CSV is loaded, so every number
   in the insight traces back to its source.
5. **Iterate.** The user reads the notebook's output and proposes changes:
   refine the query (back to step 2, often the same file), add a chart,
   change the grain, or push the insight into a decision. Each round is one
   query file, one result, one notebook delta.

## Working rules

- **Never connect to the warehouse — no exception.** Even when the user's
  machine holds working credentials, agent-to-warehouse access is prohibited.
  The query file plus the user's run is the only path.
- **Partition filters are for tables that need them.** A large table is read
  through its partition filter once the column is known; a small table needs
  no filter at all. If the size is unknown and the table may be large, close
  that hole before running anything expensive. The filter prunes only the
  table it names, so each large table in a join gets its own.
- **Write queries for the reader who runs them**: fully-qualified table paths
  copied from the profiles, the workspace's dialect, no invented columns. A
  column that is not in the index is a question, not a guess.
- **Small results paste, big results land.** The pasted table is for looking;
  the downloaded CSV is for keeping. When in doubt, ask which the user wants.
- **Notebooks read files, not networks.** The CSV in the repo is the
  notebook's input, so the insight is reproducible offline and the notebook
  never becomes a hidden query path — and the SQL behind each CSV is logged
  in the notebook where that CSV is loaded.
- **Never commit.** The user reviews the tree with `git status` / `git diff`
  and commits when satisfied.
- Measure the loop honestly at each close: queries written, results
  returned, CSVs landed, one notebook and its delta.