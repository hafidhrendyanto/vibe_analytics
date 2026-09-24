# SQL Patterns: how this pairing writes queries

The patterns below are distilled from production analytics SQL (large
attribution and dashboard pipelines), validated against real cost and
runtime. Every query written in the pairing loop follows them; when a
template from the probe set is used, these patterns apply on top.

## 1. The WITH chain reads like a program

Write the query as a pipeline of named stages: comma-joined `WITH` clauses,
one CTE per step, each named for the rows it holds, with the final `SELECT`
at top level, not indented inside the `WITH`:

```sql
WITH sessionized_click_events AS (
    ...
), product_by_variant_mapping AS (
    ...
)
SELECT ...
FROM ...
```

The `WITH` block is the program; the final `SELECT` is its entry point. A
reader starts at the bottom, sees which stage produces the answer, and climbs
to any stage's definition by name. Each CTE is a step that would be
defensible on its own: `impression_events` → `click_events` →
`add_to_cart_events` → `purchase_events` reads as the funnel it is. When a
step is reused across branches, it is already a named stage: no copy-paste,
no drift.

A nested `WITH` inside a CTE is fine, and preferred when the outer stage is
complex enough to have stages of its own: the headline pipeline stays flat,
while each stage carries its own local sub-pipeline. The inner CTEs follow
the same naming discipline and are scoped to their stage, so they cannot
collide with the outer program's names.

## 2. Name every CTE, table, and column alias explicitly

Names are the program's prose. A CTE is named after the rows it holds
(`sessionized_click_events`, not `t1` or `cte_final`); a table gets a full
alias even when it is the only table (`FROM \`project.dataset.api_request_logs\`
AS request_logs`); a column alias says what it carries
(`JSON_VALUE(event_properties, '$.search_id') AS search_id`).
Single letters are reserved for the most local contexts. The test: grep any
name and the definition appears once, obviously.

## 3. DECLARE the windows at the top

Scripting variables at the top of the file act as global parameters every
stage shares: the date windows above all, plus any constant the query
needs:

```sql
DECLARE end_date   DEFAULT DATE_SUB(CURRENT_DATE("UTC"), INTERVAL 1 DAY);
DECLARE start_date DEFAULT DATE_SUB(CURRENT_DATE("UTC"), INTERVAL 60 DAY);
```

Keep alternates as commented lines, so a window change is a two-line swap
with zero logic edits:

```sql
-- DECLARE start_date DEFAULT DATE_SUB(end_date, INTERVAL 60 DAY);
-- DECLARE start_date DEFAULT DATE("2026-01-01");
```

The same block carries constants (`DECLARE radians_per_degree DEFAULT
ACOS(-1) / 180; -- BigQuery has no RADIANS()/PI() builtin`) and list
parameters (`DECLARE target_regions DEFAULT [...]`). When the pairing
iterates a query, only the `DECLARE` block moves.

## 4. CREATE TEMP TABLE when a stage is reused across branches

Optional, but decisive for big pipelines. A CTE referenced several times is
recomputed per reference; a temp table is evaluated exactly once, so the
shared scan runs once:

```sql
CREATE TEMP TABLE base_events AS (
  SELECT ... FROM `<project>.<dataset>.fact_events` ...
);
```

A dashboard query with a dozen output branches defines the shared stage once
and reuses it across all of them. Two costs to weigh:

- **Every read of a temp table is billed.** Prefer loading the temp table
  already aggregated (or column-pruned to exactly what consumers project) so
  the repeated reads pay for small rows, not raw events. In one production
  pipeline, carrying a raw JSON payload column through the chain accounted
  for about two thirds of the run's bytes; parsing it once in the base stage
  and dropping it cut the total by more than half.
- **The payoff is wall time.** Many organizations cancel queries past a hard
  runtime limit; materializing the shared stages is what keeps a
  many-branch rollup inside it. When a single-pass query fits comfortably in
  the limit, skip the temp tables: the plain `WITH` chain is simpler and
  bills nothing extra.

Rule of thumb: plain `WITH` chain until a stage is referenced more than once
or the runtime approaches the limit; then materialize the shared stages, and
aggregate them first.

## 5. Partition when necessary

Filter on the **declared** partition column, and let the profiles say which
one it is: the metadata view can report nothing on views while the backing
table prunes fine. A request-log table may be partitioned on
`response_time` while `created_at` merely tracks it closely and happens to
prune almost as well; pair both columns when the mapping needs both, but
keep the pruning predicate on the declared column.

Clustered columns prune scans the same way: on an events table clustered by
`event_type`, restricting `event_type IN (...)` to exactly the types the
logic inspects reads only those blocks. Column pruning is the same economy
at the projection level: BigQuery bills the projected bytes of every read
(temp reads included), so select only what a downstream stage consumes and
drop dead columns at each hop.

## The supporting patterns

- **Header banner.** Open the file with the `-- =====` banner: what it
  answers, which variant it is, the cost story, and any deliberate deviation.
  A good header is what lets the file be read cold.
- **`WHERE TRUE` + `AND` chains.** Starting every filter list with `TRUE`
  keeps every predicate a grep-able `AND` line, and commenting one out is
  never a syntax edit.
- **`CREATE TEMP FUNCTION` for shared CASE logic**: `normalize_channel(source, medium)`
  keeps a many-way mapping in one place instead of three copies of the CASE.
- **Comments explain why, not what.** The partition note inside a WHERE
  ("response_time is this table's declared partition column: filtering on it
  is what actually prunes") is worth more than a restatement of the SQL.
- **Push-down reads.** The base stages select only the columns their
  consumers project and carry the filters down to the scan; the widest base
  filter plus per-stage filtering downstream beats re-scanning per branch.
