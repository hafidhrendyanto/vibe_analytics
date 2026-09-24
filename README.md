# vibe_analytics

Skills for working with a BigQuery data warehouse **through pairing**: one
skill builds the documentation index of the tables your SQL work touches,
the other uses that index to draft, run, and refine queries with you.

Both skills share one standing rule, stated as a hard rule in each: **the
agent never connects to the warehouse.** Every query is written by the agent
to a file, run by you in your analytics workspace (e.g. Mode Analytics), and
the result comes back through you — pasted for a quick look, or downloaded
into the repo when it feeds a notebook. The skills assume a BigQuery
warehouse and are written in its dialect.

## The skills

| Skill | What it does |
|-------|--------------|
| [`create-dataset-index`](skills/create-dataset-index/SKILL.md) | Builds a `datasets/` documentation tree: a README index plus one profile per table (`tables/<project-slug>/<dataset>/<table>.md`). Mines a repository full of SQL files, or seeds from tables and queries you name in chat, then closes knowledge gaps through an iterative probe loop. |
| [`pair-analytics`](skills/pair-analytics/SKILL.md) | Uses the index to pair on analytics: turns a natural-language request into a correct, cheap query written to `datasets/analytics/`, gets the result back through you (pasted or as a downloaded CSV), and proposes a notebook — marimo preferred, Jupyter fine; polars preferred over pandas — that shows the insight from the downloaded file. |

The two compose: the first builds the knowledge, the second spends it.

## Install

You do **not** need to clone the repo yourself first. The `skills` CLI
(skills.sh, the installer of the open agent-skills ecosystem) fetches the
GitHub repository named `<owner>/vibe_analytics` on its own — replace
`<owner>` with the GitHub account that hosts this repo — then scans it
recursively for `SKILL.md` directories, lets you pick which skills to
install, and links or copies them into the supported coding agents
(symlink by default; `--copy` for copies). Useful flags:

```bash
npx skills add <owner>/vibe_analytics
```

| Flag | Behavior |
|------|----------|
| `-g, --global` | Install to the user directory instead of the project |
| `-a, --agent <agents...>` | Target specific agents (e.g. `-a claude-code`) |
| `-s, --skill <skills...>` | Install specific skills by name (`'*'` for all) |
| `-l, --list` | List available skills without installing |
| `--copy` | Copy files instead of symlinking |

Two preconditions for the one-liner: the repo must actually be on GitHub
under that `<owner>` (public; private repos need your own GitHub auth), and
it is not there yet — until it is pushed, use the manual install below. The
CLI also has not been verified against `pi` as a target; if it does not
detect pi in your setup, install the skills into pi's directories directly:

| pi scope | Location | Command |
|----------|----------|---------|
| Project | this repo's `.pi/skills/` | `mkdir -p .pi/skills && cp -r skills/<skill-name> .pi/skills/` |
| Global | `~/.agents/skills/` | `cp -r skills/<skill-name> ~/.agents/skills/` |

After installing in a live pi session, run `/reload`; the skills then load
automatically when a task matches, and are also available as
`/skill:<skill-name>`. For Claude Code, restart the session so the skills
are discovered.

| Flag | Behavior |
|------|----------|
| `-g, --global` | Install to the user directory instead of the project |
| `-a, --agent <agents...>` | Target specific agents (e.g. `-a claude-code`) |
| `-s, --skill <skills...>` | Install specific skills by name (`'*'` for all) |
| `-l, --list` | List available skills without installing |
| `--copy` | Copy files instead of symlinking |

If `pi` is not among the CLI's supported targets in your setup, install the
skills into pi's directories manually:

| pi scope | Location | Command |
|----------|----------|---------|
| Project | `datasets` repo's `.pi/skills/` | `mkdir -p .pi/skills && cp -r skills/<skill-name> .pi/skills/` |
| Project (Agent Skills) | `.agents/skills/` | `cp -r skills/<skill-name> .agents/skills/` |
| Global | `~/.agents/skills/` | `cp -r skills/<skill-name> ~/.agents/skills/` |

After installing in a live pi session, run `/reload`; the skills then load
automatically when a task matches, and are also available as
`/skill:<skill-name>`. For Claude Code, restart the session so the skills
are discovered.

> Review the skills before installing them in a project scope: both read and
> write files inside your project and, per their hard rule, never touch the
> warehouse or the network.

## What each skill does, end to end

**`create-dataset-index`** — the documentation builder. It mines the SQL
your repo (or your pasted queries) references, writes a `datasets/README.md`
index plus one profile per table, and closes knowledge holes (column types,
meanings, partition columns) through an iterative probe loop: it writes
copy-pasteable probe queries, you run them in your analytics workspace and
paste results back, and each answer lands in the profiles with its source
and date. The loop's ledger lives in the conversation; the profiles on disk
are the durable record, so you can pause into another session at any time.

**`pair-analytics`** — once the index exists, pairs with you on analytics:
restates the request as a query plan, writes the query into
`datasets/analytics/<child>/`, you run it in Mode Analytics, and the result
comes back through Mode's own result actions — copied into the chat for
debugging and stepping stones, or downloaded as a CSV into the repo when it
is large or notebook-bound. A landed CSV earns a proposed notebook that
reads only that file and shows the insight, so every number traces back to
its source query, which the notebook logs.

## License

MIT.
