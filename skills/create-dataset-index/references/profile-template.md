# Profile Template — one table's profile

The shape every table profile follows. One file per table at
`datasets/tables/<project-slug>/<dataset>/<table>.md`. Fill every placeholder
from what the build learned; a section with nothing known yet stays as a
visible hole rather than being filled with a guess. The house rules in
`SKILL.md` govern what each section may contain.

````markdown
# `<dataset>.<table>`

- **Full path**: `<project_id>.<dataset>.<table>`
- **Type**: TABLE / VIEW / MATERIALIZED VIEW / EXTERNAL / SNAPSHOT / CLONE / WILDCARD (<member type>)

## Purpose

One-line description of what this table contains.

## Business purpose

Why the team keeps this table in its workflows; the motivation before the
mechanics.

## Key fields

- `field1` (TYPE) — description
- `field2` (TYPE) — description

## First seen in

`<path/to/reviewed_file.sql>`

## Used in

`<path/to/other_file.sql>`, ...

## Example values

Five sample rows (sample-rows probe, <date>):

| `field1` | `field2` | `field3` |
|---|---|---|
| `value` | `value` | `value` |
| `value` | `value` | `value` |
| `value` | `value` | `value` |
| `value` | `value` | `value` |
| `value` | `value` | `value` |

For a column holding JSON (or any value too big for a table cell), the cell
reads `(JSON, row <n> below)` and the full value follows, one block per row:

`field3`, row 1:

```json
{
  "key": "value"
}
```

<N> rows (COUNT probe, <date>)

## Partitioning

- **Declared** (DDL probe, <date>): `PARTITION BY <expr>`, `CLUSTER BY <cols>`,
  or "view of `<backing_table>`"; "not readable" if the dataset denies metadata.
- **Measured** (dry run, <date>): unfiltered <N> GB vs filtered on `<column>`
  <M> GB.

## Relationships

- Joined with `<other_table>` on `<key>` via `<join_type>`
````
