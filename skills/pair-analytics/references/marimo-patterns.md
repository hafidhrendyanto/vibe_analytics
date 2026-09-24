# marimo Patterns — writing the notebook as a file

How to write a marimo notebook as a `.py` file when no live marimo session is
running. With a live session, drive it through the `marimo-pair` skill
instead: the kernel owns the file then, and edits made on disk never reach
it.

marimo notebooks are plain Python files, so a type checker (Pylance, pyright)
checks them like any module, and marimo rewrites them on every save. The
rules below keep a notebook type-clean and stable across saves.

## 1. Every import goes in `app.setup`

A regular cell's variables reach downstream cells as **function
parameters**. An import made in a cell therefore arrives as a parameter, and
using it in an annotation is a type error (`Variable not allowed in type
expression`):

```python
# WRONG: mo and pl are parameters, so both annotations fail
@app.cell
def _():
    import marimo as mo
    import polars as pl
    return mo, pl

@app.cell
def _(mo, pl):
    summary: pl.DataFrame = ...
```

`app.setup` runs at module level, so its names are real modules:

```python
# RIGHT
with app.setup(hide_code=True):
    import marimo as mo
    import polars as pl

@app.cell
def _():
    summary: pl.DataFrame = pl.read_csv("data/summary.csv")
    return (summary,)
```

Put **every** import there (stdlib, third-party, first-party), along with
module-level setup such as `matplotlib.use("Agg")` or palette constants. A
setup name must never appear in a cell signature.

## 2. Annotate the declaration, never the signature

marimo owns each cell's `def _(...)` line and regenerates it on save.
Annotate where a variable is **declared**; marimo copies the type into every
consumer's signature:

```python
@app.cell
def _():
    orders: pl.DataFrame = pl.read_csv("data/orders.csv")  # annotate here
    grain: mo.ui.dropdown = mo.ui.dropdown(["Daily", "Weekly"], value="Daily")
    return grain, orders
```

- **A cell's variable is visible downstream only if the cell returns it.**
  When adding a widget or a frame, add it to the `return` tuple in the same
  edit, or the reading cell fails with a `NameError`.
- **Return only what a later cell reads.** marimo rewrites the return to
  match on save; matching it up front keeps diffs quiet.
- **Pure helpers go in `@app.function`**: module level, annotated normally,
  importable.

## 3. What survives a save

- **Comments between cells are dropped**, including the line right above an
  `@app.cell`. Put explanations inside the cell body or in a docstring.
- **The module docstring and top-of-file comments are kept.**
- **Signatures, parameter order and `return` tuples are regenerated.**
  Parameters come back sorted alphabetically.

## 4. Rendering and stopping

- **A cell's bare last expression is its rendered output.** A type checker
  flags it as `reportUnusedExpression`; that is expected. Never delete it,
  or the cell renders nothing.
- **`mo.stop` does not narrow an Optional for the type checker.** Follow it
  with an `assert` that can never fire:

  ```python
  mo.stop(result is None, mo.md("Click **run** first."))
  assert result is not None  # narrows the type for the lines below
  ```

## 5. Skeleton for an analysis notebook

The shape for a notebook that reads a downloaded CSV. It loads each CSV in
its own cell, with a pointer to the query file behind it, so every number
traces back to its SQL:

```python
"""<Notebook title>: what question it answers, from which result files."""

import marimo

__generated_with = "0.24.2"
app = marimo.App(width="full")

with app.setup(hide_code=True):
    import marimo as mo
    import polars as pl


@app.cell(hide_code=True)
def _():
    mo.md(r"""
    # <Notebook title>

    What it shows, where the data comes from, what to read off it.
    """)
    return


@app.cell
def _():
    # Source: datasets/<topic>/<query>.sql, run in Mode on <date>.
    orders: pl.DataFrame = pl.read_csv("datasets/<topic>/<result>.csv")
    return (orders,)


@app.cell
def _(orders: pl.DataFrame):
    daily = orders.group_by("order_date").agg(pl.len().alias("orders"))
    daily
    return


if __name__ == "__main__":
    app.run()
```

## 6. Verify before handing it over

A clean editor panel is not proof. Check it headlessly:

```bash
marimo check <notebook>.py   # marimo's own lint; expect exit 0
python <notebook>.py         # script mode: runs every cell end to end
npx -y pyright <notebook>.py # optional; expect only reportUnusedExpression
```

Script mode catches what type checking cannot: an unreturned variable, a
stale signature, a wrong path to the CSV. Run it before saying the notebook
works.
