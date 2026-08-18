# CLAUDE.md

Guidance for AI assistants working **on the hyperresearch codebase**. This is the
tool's own repository, not a research vault.

> **Do not confuse the two roles of this filename.** In a *user's* project,
> `CLAUDE.md` is a generated artifact: `hyperresearch install` / `hyperresearch init`
> writes the research-workflow blurb into it (see `core/agent_docs.py`). In *this*
> repo, `CLAUDE.md` is a hand-written contributor guide that is tracked in git.
> If you run `hyperresearch install .` or `hyperresearch init .` inside a checkout,
> `inject_agent_docs()` will **append** its `<!-- hyperresearch:start -->` block
> below this text. That appended block must never be committed —
> `git checkout -- CLAUDE.md` to drop it. Prefer installing into a scratch
> directory (`hyperresearch install /tmp/hpr-smoke`) instead of the checkout.

## What this project is

`hyperresearch` is a **Claude Code harness** distributed as a Python package. It
has two halves that are developed and reviewed differently:

1. **A Python CLI + vault engine** (`typer` CLI, SQLite/FTS5 index, markdown notes,
   web fetchers). Ordinary Python: tested, linted, type-annotated.
2. **A prompt payload** — 17 markdown skill files in `src/hyperresearch/skills/`
   plus ~2,900 lines of subagent prompt templates embedded as string constants in
   `src/hyperresearch/core/hooks.py`. These are *shipped data*, not code. They
   define the 16-step research pipeline that Claude Code executes.

`hyperresearch install` copies half 2 into a target project's `.claude/`
(skills + agents + a `PreToolUse` hook) and initializes a vault. Users then run
`/hyperresearch <query>` in Claude Code. The Python CLI is the tool surface those
agents call — every command supports `--json` / `-j` because its primary consumer
is an LLM, not a human.

Read `README.md` for the pipeline overview and
`src/hyperresearch/skills/hyperresearch.md` (the entry-skill router) for the
authoritative pipeline contract. **When the skill files and prose docs disagree,
the skill files win.**

## Repository layout

```
src/hyperresearch/
  cli/            typer commands — one module per command/sub-app; registered in cli/__init__.py
  core/           vault, config, SQLite schema+migrations, sync engine, frontmatter,
                  note IO, fetcher, linker, similarity, templates, patterns (wiki-link regexes),
                  hooks.py (subagent prompt templates + installer), agent_docs.py (CLAUDE.md injector)
  models/         pydantic/StrEnum schemas — note.py (frontmatter vocab), output.py (Envelope), search.py, graph.py
  search/         FTS5 query preprocessing, ranking, filters
  graph/          (namespace only — link parsing lives in core/patterns.py + core/linker.py)
  indexgen/       auto-generated index pages (_index, _tags, _recent, _orphans, _stats)
  web/            pluggable fetch/search providers: builtin, crawl4ai, exa, tavily (base.py = Protocol + WebResult)
  serve/          minimal read-only web UI (stdlib http.server)
  mcp/            optional MCP server exposing 8 read-only vault tools
  export/         (namespace only — export logic lives in cli/export.py)
  skills/         the 17 shipped skill markdown files (entry router + 16 steps)
tests/            pytest, mirrors src layout (test_cli, test_core, test_web, test_search, test_graph)
assets/           README banner + benchmark chart, plus the scripts that generate them
example-reports/  a real pipeline output, committed as a sample
.github/workflows/ci.yml (ruff + pytest + build, py3.11–3.13), publish.yml (tag → PyPI)
```

Note the vocabulary mismatch to expect: the CLI/package is `hyperresearch` (alias
`hpr`); the *user-facing* content directory is `research/`; the *hidden* state
directory is `.hyperresearch/` (config.toml, hyperresearch.db, hook.js, templates,
exports). None of those exist in this repo — they're gitignored per-user artifacts.

## Development workflow

```bash
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"

pytest tests/ -q              # 239 tests, ~10s. Must stay green.
ruff check src/ tests/        # must be clean — CI runs exactly this
ruff format src/              # line length 100
python -m build               # CI also builds the wheel; keep packaging working
```

**Verified state as of v0.8.7:** `pytest` 239 passed, `ruff check src/ tests/` clean.

**mypy is configured strict but is NOT enforced and does NOT pass** —
`mypy src/hyperresearch/` reports ~168 errors, concentrated in `mcp/server.py`
(30), `serve/server.py` (27), `web/crawl4ai_provider.py` and `cli/note.py` (9
each). CI (`.github/workflows/ci.yml`) runs only ruff, pytest, and build. Don't
"fix mypy" as a side quest inside an unrelated change, and don't trust
`CONTRIBUTING.md`'s claim that strict mode is a gate. Do keep new code annotated.

**`CONTRIBUTING.md` is stale.** It was written for a predecessor project called
`kasten` and still names `kasten` commands and modules that don't exist here
(`src/kasten/ingest/`, `compile/`, `llm/`). Trust this file and the actual tree
over it.

### Git / release conventions

- Conventional-commit prefixes with a PR reference:
  `feat: …(#43)`, `fix(web): …(#36)`, `docs: …`, `ci: …`, `lint: …`, `release: v0.8.7 (#45)`.
- Version lives in **two** places that a test enforces: `pyproject.toml`
  `[project].version` and `src/hyperresearch/__init__.py` `__version__`.
  `tests/test_core/test_packaging.py::test_runtime_version_matches_project_metadata`
  fails if they drift.
- `CHANGELOG.md` is maintained per release with narrative entries (what broke, who
  reported it, what the fix was). Add an entry for any user-visible change.
- Pushing a `v*` tag publishes to PyPI via `publish.yml` (which re-runs ruff +
  pytest before uploading).

## Code conventions

Every CLI command follows the same shape. Match it exactly:

```python
def vault_tag(
    slug: str = typer.Argument(..., help="..."),
    json_output: bool = typer.Option(False, "--json", "-j", help="JSON output"),
) -> None:
    """One-line docstring — this is the --help text."""
    from hyperresearch.core.vault import Vault, VaultError   # imports INSIDE the function

    try:
        vault = Vault.discover()
    except VaultError as e:
        if json_output:
            output(error(str(e), "NO_VAULT"), json_mode=True)
        else:
            console.print(f"[red]Error:[/] {e}")
        raise typer.Exit(1)
    ...
    if json_output:
        output(success(data, count=len(data), vault=str(vault.root)), json_mode=True)
    else:
        console.print(...)  # rich, human-readable
```

- **Heavy imports go inside command functions**, not at module top. CLI startup
  time matters because agents shell out dozens of times per run.
- **Dual output is mandatory.** `--json` emits the `Envelope`
  (`ok`/`data`/`error`/`error_code`/`count`/`vault`/`timestamp`, see
  `models/output.py`) via `cli/_output.py:output()`. Human mode uses `rich`.
  Never `print()`.
- **Error codes are strings** (`NO_VAULT`, `INVALID_SLUG`, `BAD_QUERY`,
  `TAG_SPACE_EXHAUSTED`) and are part of the agent-facing contract. Exit non-zero
  on failure (`typer.Exit(1)`; search uses `2` for `BAD_QUERY`).
- **Never silently return empty.** A malformed query or a failed fetch must be
  distinguishable from "no results" — see `search/fts.py:SearchQueryError` and the
  PDF-diagnostics logging in `core/fetcher.py`. This is a load-bearing lesson from
  #32 and #39; regressing it is a bug.
- `from __future__ import annotations` at the top of every module.
- ruff select `E,F,I,N,W,UP,B,SIM,RUF`; `E501` ignored but keep lines near 100.

## Data model

**Markdown is truth, SQLite is a rebuildable cache.** Notes are markdown +
YAML frontmatter under `research/notes/`; the index at
`.hyperresearch/hyperresearch.db` can be deleted and rebuilt with
`hyperresearch sync --force`. Never make SQLite the only home for anything.

Controlled vocabularies are declared **twice** and must stay in sync:
`models/note.py` StrEnums (`NoteStatus`, `NoteType`, `Tier`, `ContentType`) and
SQLite `CHECK` constraints in `core/db.py:SCHEMA_SQL`. Adding a value means:

1. Extend the StrEnum in `models/note.py`.
2. Update `SCHEMA_SQL` in `core/db.py` (for fresh vaults).
3. Bump `SCHEMA_VERSION` in `core/db.py`.
4. Add an idempotent entry to `MIGRATIONS` in `core/migrations.py` (SQL string, or
   a callable when you need conditional logic). SQLite can't alter a `CHECK` in
   place — follow the table-rebuild pattern in `_migrate_v7_interim_note_type` /
   `_migrate_v8_source_analysis_note_type`, including the cheap probe that makes
   re-running a no-op.

Migrations run on **every vault open** (`Vault.db` → `init_schema` → `migrate`),
so they must be cheap and idempotent. Indexes on migration-added columns belong in
`POST_MIGRATE_INDEXES_SQL`, not `SCHEMA_SQL`.

Sync gotchas worth preserving (both are fixes for real data-loss bugs, #25):
`compute_sync_plan` skips files directly at the `research/` root (staging files
like `scaffold.md`) and probes the first 16 bytes to skip anything lacking a YAML
frontmatter delimiter (agent scratch files); `execute_sync` refuses to UPSERT a
note id already owned by a different path.

## Wiring checklists

**New CLI command** → module in `cli/`, then register in `cli/__init__.py`
(`app.command("name")(fn)` for a root command, `app.add_typer(sub_app, name=...)`
for a group). Add a `CliRunner` test in `tests/test_cli/`.

**New pipeline step skill** → markdown file in `src/hyperresearch/skills/` named
`hyperresearch-N-slug.md`, add its name to `_HYPERRESEARCH_STEP_SKILLS` in
`core/hooks.py`, and update the routing tables in `skills/hyperresearch.md` (the
entry router), the README table, and the tier routing table. Hatchling ships
everything under `src/hyperresearch/`, so no packaging change is needed — but
verify (`python -m build` then check the wheel lists 17 `skills/*.md`). Skill dirs
matching `hyperresearch-*` that aren't in the roster get pruned from users'
`.claude/skills/` on the next install, so renames clean themselves up.

**New subagent** → add the prompt template constant to `core/hooks.py`, add an
`_install_<name>_agent()` helper using `_write_agent_file`, and register it in
**both** `install_hooks()` and `install_global_hooks()`. Retired agents go in
`_RETIRED_AGENT_FILES` so upgrades prune them. Keep the roster table in `README.md`
accurate (it has drifted before — see the 0.8.4 changelog).

**New lint rule** → add the id + description to `RULES` in `cli/lint.py` and
implement the check in the same function; severities are `error` / `warning` /
`info`, and `error` blocks the pipeline's final integrity gate. Scaffold-section
detection must use `SCAFFOLD_ONLY_SECTION_HEADERS` from `core/hooks.py` — the
single source of truth shared by critics, the polish auditor, and lint.

**New web provider** → implement the `WebProvider` Protocol in `web/base.py`
(`name`, `fetch`, `search`), add a branch to `get_provider()`, declare an optional
extra in `pyproject.toml`, and add the SDK to the `dev` extra if you write tests
(`tests/test_core/test_packaging.py` asserts dev covers tested optional
providers — CI installs only `.[dev]`). Provider tests stub the SDK; no network in
the test suite.

## Traps

- **Tool-lock invariant.** The patcher and polish auditor are restricted to
  `[Read, Edit]` in their agent frontmatter so they physically cannot regenerate a
  report. Never add `Write` to those agents; never loosen "patch, never
  regenerate" in the step skills.
- **Prompt text is a shipped interface.** Editing a skill file or an agent template
  changes runtime behavior for every user on the next install, with no test
  coverage to catch it. Read the surrounding contract (spawn contract, tier gate,
  invariants list) before touching it, and keep the numbered artifact paths in
  `skills/hyperresearch.md`'s recovery section consistent with what steps write.
- **Absolute paths in generated output.** `agent_docs.py:_resolve_executable()`
  bakes the installing user's binary path into their `CLAUDE.md`. That's why
  install outputs (`.claude/`, `AGENTS.md`, `GEMINI.md`) are gitignored.
- **Python 3.11–3.13 only** (`requires-python = ">=3.11,<3.14"`). 3.14 breaks on
  Crawl4AI's `lxml` pin; `cli/__init__.py` warns at startup. Windows console
  encoding is also patched there — don't reorder those pre-import blocks.
- **`research/` and `.hyperresearch/` are gitignored**, as are `bench/`,
  `codex_bench/`, `draco-bench/`. Don't commit vault output from a local test run.
- Wiki-link parsing has a large false-positive filter in `core/patterns.py`
  (citation footnotes, roman numerals, figure refs) because crawled markdown emits
  `[[100]]`-shaped noise. Extend `is_valid_wiki_link_target` with a test rather
  than loosening the regexes.
