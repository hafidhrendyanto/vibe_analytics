---
name: pair-analytics
description: >
  Pair with the user on warehouse analytics through the project's datasets
  index: agree on a plan, check its assumptions with small probe queries the
  user runs in their analytics workspace (Mode Analytics), and only then write
  the correct, cost-efficient final BigQuery query to a file — the agent never
  connects to the warehouse. The agent stops after every hand-over, so the
  user steers each step. Probe results come back pasted into the chat; final
  results come back as a downloaded CSV in the repo, followed by a proposed
  notebook (marimo preferred, Jupyter fine) that shows the insight. Use when
  asked to write or refine warehouse queries, pull data for analysis, explore
  indexed tables, or turn a data question into a notebook.
license: MIT
metadata:
  version: "0.1.0"
---

# Pair Analytics

Pair with the user on warehouse analytics: turn what they ask for in natural
language into a correct, cost-efficient BigQuery query, get the result back
through them, and turn the result into an insight they can react to.

**A wrong assumption corrected at the start is worth a thousand lines of
code.** This is a pairing, not a delegation. The user does more than run
queries: they point the direction. They know which rows really count as a
customer, which event is really a click, and which number looks wrong. So the
agent moves in small steps and stops after each one. It never runs ahead to
finish the analysis alone and leave the user to review a pile of assumptions
afterwards.

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
agent writes the query (a probe in the chat, a final query in a file), the
user runs it in their analytics workspace (Mode Analytics or similar), and the
result comes back through the user — pasted into the chat or downloaded into
the repo. The user's run is the only execution the skill has.

This rule outranks every other instruction in this skill. If any step seems to
offer a direct connection, stop and reread this section.

## What the index gives each query

Before writing any query, read the profiles of the tables the request
touches. The index answers, without guessing:

- **Full path** — the copyable `project.dataset.table` for every FROM clause.
- **Key fields** — the column names, types, and meanings that turn the
  request's words into real column names.
- **Partitioning** — the filter that keeps the scan lean. A query on a large
  table without its partition filter is the difference between a read that
  costs cents and one that costs tens of dollars; never emit one without it
  once the column is known.
- **Relationships** — the join keys the team already uses.
- **First seen in / Used in** — how the team queries these tables today; a
  refinement should start from the most cost-efficient correct existing
  shape.

If a needed fact is missing from the index (a column the profiles do not
carry, a partition column not yet verified), do not guess: probe it (step 2
of the loop) or ask the user directly. The gap becomes a hole to fill, not a
silent assumption in the query.

## The loop

Every step ends with a hand-over, and **every hand-over ends the turn**. The
agent hands over a plan, probes, or a final query, then stops and waits for
the user. When the user says "continue" while results are still pending, the
agent hands over the next pending query, or asks for the missing result, and
stops again. It does not draft later queries, final queries, or the notebook
ahead of the results they depend on: work built on an unchecked assumption
is work the user will have to unpick.

1. **Restate the request as a plan, and list its assumptions.** Give the
   tables, the filters, the grain, and the expected result shape. Then list
   what the plan takes on faith: how rows map to the entities the user named,
   which event or code means the action they named, what the scan will cost,
   and any definition the question leaves open ("repeat", "baseline"). Stop
   here. The user corrects the aim and the assumptions before any SQL is
   written.
2. **Probe the assumptions, in the chat.** A probe is a small query whose
   result settles one assumption: a coverage count, the distinct values of a
   code column, a sample of a join, a dry-run cost. Show it as a fenced SQL
   block in the chat for the user to copy into Mode. Never write a probe to
   a file: it is a stepping stone, not a keeper. Hand over **at most three
   probes per round**, so the user can run them side by side. Label each one
   with the assumption it checks and the decision its result feeds. Design
   every probe as lean as it can be (the partition filter, a narrow window,
   or a dry run) and to return at least one row, since Mode shows nothing
   for an empty result. Then stop and wait for the results.
3. **Read the results back.** For each probe, state what it showed, which
   assumption it confirmed or broke, and what that changes in the plan. When
   a result fills a hole in the index, update that table's profile (a
   Measured line, a code meaning, a coverage note) and name the probe and
   date behind it. If assumptions remain open, go back to step 2 for another
   round. Otherwise, move on.
4. **Write the final query to a file**, only once every probe it depends on
   has come back and been read. Before writing it, give its design in the
   chat and tie each assumption to the probe result behind it. One file per
   query, named after what it answers. The default home is
   `datasets/analytics/`: pick the child
   directory inside it that fits the analysis best (an existing one such as
   `datasets/analytics/retention/`, or a new child named after the
   work when none fits). Write it in the bundled SQL patterns — read
   `references/sql-patterns.md` before the first query of a session. The user
   copies the query from the file into Mode Analytics and runs it there.
   Hand over one final query per round, then stop.
5. **The result comes back through one of two channels** — Mode's own
   result actions decide which:

   - **Copy result** (clipboard, pasted into chat): for probes, small
     results, and anything that is debugging or a stepping stone toward a
     more final query. Iterate fast; nothing lands on disk yet.
   - **Download CSV** (moved into the repo): when the result is large, or it
     is going to be used in a notebook. Ask the user where to put it if the
     project has no convention; a predictable home inside the same
     `datasets/analytics/` child keeps the data next to the query and the
     notebook that reads it.

   State the recommendation, then ask — the user knows whether the result is
   a stepping stone or a keeper.

6. **Propose the notebook — or the quick answer.** When a CSV lands, propose a
   notebook that reads it and shows the insight. Never start the notebook
   before the CSV exists and the user has agreed to it. The user can see the
   data and propose new changes from there. But when a quick answer is needed,
   or the user asks for one, answer in chat instead — a short TL;DR with a
   markdown table (or a small chart sketch) beats a notebook for a question
   that only wants answering; the notebook can follow once the analysis
   grows. Ask two questions before creating one:

   - **Marimo or Jupyter?** Marimo is preferred; Jupyter is fine when the
     user's stack prefers it.
   - **Polars or pandas?** Polars is preferred; pandas is fine when the user
     asks for it.

   For marimo, how the notebook gets written depends on whether it is open
   in a live session:

   - **A live marimo kernel is running** (the user has `marimo edit` open):
     drive it through the `marimo-pair` skill. The kernel owns the file
     while the session runs, so a file edit never reaches the user and may
     be overwritten on the next save.
   - **No live kernel:** write or edit the `.py` file directly, following
     `references/marimo-patterns.md`. Before handing it over, run the
     checks in that file.

   The notebook reads the downloaded CSV — never queries the warehouse, never
   calls the network. It is the lens on the result, not a second execution
   path. And it logs the query behind each CSV it reads: the SQL itself or a
   pointer to its query file, right where the CSV is loaded, so every number
   in the insight traces back to its source.
7. **Iterate.** The user reads the notebook's output and proposes changes:
   refine the query (back to step 4, often the same file; back to step 2
   when the change rests on a new assumption), add a chart, change the
   grain, or push the insight into a decision. Each round is one query file,
   one result, one notebook delta.

## Working rules

- **Never connect to the warehouse — no exception.** Even when the user's
  machine holds working credentials, agent-to-warehouse access is prohibited.
  The query file plus the user's run is the only path.
- **Probe before final.** A final query is written only after every probe
  it depends on has returned and its finding has been stated to the user. A
  final query drafted "to save time" while probes are pending bakes in the
  very assumptions the probes exist to check.
- **Stop after every hand-over.** A plan, a probe round (at most three
  probes), or a final query ends the turn. "Continue" means "hand me the
  next thing", not "finish without me".
- **Probes live in the chat; finals live in files.** A probe is shown as a
  fenced SQL block to copy into Mode, never saved. Only a final query gets a
  file under `datasets/`.
- **Say where every number comes from.** When citing a cost, a row count, or
  a fact, name its source: "from the index profile's Measured line, dry run
  of <date>", "from probe 2's result", "from the user". Never phrase a number
  as if the agent measured it; the agent has no warehouse access, and the
  user should never have to wonder whether it quietly ran something.
- **Cost-efficient, not cheap.** A query may cost what the question
  justifies: a six-month scan is fine when the answer needs six months. The
  goal is the most cost-efficient query for that answer (partition and
  cluster filters, pruned columns, aggregated temp tables), never a smaller
  question to keep the bill low. State the expected cost with the hand-over,
  and flag a large one so the user can decide before running it.
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
- Measure the loop honestly at each close: probes run, assumptions settled,
  queries written, results returned, CSVs landed, one notebook and its delta.
