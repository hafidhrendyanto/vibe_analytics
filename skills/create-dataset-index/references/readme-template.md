# README Template — the datasets index

Fill every placeholder from what the build learned; delete optional sections
the project does not need. The bracketed comments inside the code block are
guidance and do not belong in the final README.

````markdown
# Datasets & BigQuery Tables Index

> Human-readable index of the warehouse tables this repo's SQL work touches.
> For detailed per-table profiles, see [`tables/`](./tables/).

## How to Discover a Table Schema

Use this query template to inspect any BigQuery table's columns, types, and
descriptions (paste into the analytics workspace):

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

## How to Check a Partition Column

The dry-run function returns a query's scan estimate without executing it.
Run the unfiltered and filtered calls and compare the reported bytes:

```sql
SELECT `functions.dry_run`(
"""
SELECT *
FROM `<project>.<dataset>.<table>`
"""
)
```

Replace the inner query with one filtered on the suspected partition column;
a large byte drop confirms it. Record both numbers with their date in the
table's profile.

---

## Project ID Mapping

Profile paths use short slugs; this table resolves them to real projects.

| Slug | Real BigQuery Project ID |
|------|--------------------------|
| `<slug>` | `<project-id>` |

Profile files live at `datasets/tables/<slug>/<dataset>/<table>.md`.

---

## How to Find a Detailed Table Profile

Given a fully-qualified table like `<project_id>.<dataset>.<table>`:

1. Look up the project slug from the mapping table above → `<slug>`
2. Open `datasets/tables/<slug>/<dataset>/<table>.md`

---

## Quick Reference — All Tables

| Table | Type | Purpose | Profile |
|-------|------|---------|---------|
| `<dataset>.<table>` | TABLE | One-line description of what it holds | [table.md](./tables/<slug>/<dataset>/<table>.md) |

The index is the directory: Table, Type, one-line Purpose, and the profile
link. Everything else — provenance, costs, investigations — lives in the
profile, so facts exist in one place and do not drift between layers.
````
