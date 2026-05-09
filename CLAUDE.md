# CLAUDE.md

## Project Context

DimOS is an agentic operating system for generalist robotics. The codebase is centered on:

- `Modules`: autonomous processes communicating through typed streams.
- `Blueprints`: runnable robot stacks composed from modules.
- `Skills`: agent-callable robot capabilities exposed through `@skill`.

Most implementation work should follow the architecture and examples in `AGENTS.md`.

## Claude Code Role

Codex is the primary coding and review agent for this repository. Claude Code, Minimax, Kimi,
or a local Hermes runner may be used as optional auxiliary agents for implementation, comparison,
or deeper review. When Claude Code is used, follow the same project constraints as Codex and keep
patches small and focused.

## Working Rules

- Read the relevant files before editing; do not rewrite architecture from memory.
- Prefer small, focused diffs that match existing DimOS patterns.
- Do not modify unrelated files, generated files, or formatting outside the touched area.
- Add or update tests for behavior changes and bug fixes.
- For bug fixes, explain the root cause and why the fix addresses it.
- Do not add production dependencies unless the task clearly requires them.
- Do not edit `dimos/robot/all_blueprints.py` manually; regenerate it with the documented test.
- For `@skill` methods, include docstrings, type annotations, and useful `str` return values.
- Treat hardware, networking, auth, data deletion, process lifecycle, concurrency, and robot motion as high-risk areas.

## Validation Commands

Install dependencies:

```bash
uv sync --extra all
```

Fast test loop:

```bash
./bin/pytest-fast
```

Targeted test:

```bash
pytest -sv dimos/path/to/test_file.py
```

Full non-tool CI-style test set:

```bash
./bin/pytest-slow
```

Type check:

```bash
uv run mypy dimos/
```

Formatting and linting:

```bash
uv run ruff format dimos tests scripts
uv run ruff check dimos tests scripts
```

After adding or renaming blueprints:

```bash
pytest dimos/robot/test_all_blueprints_generation.py
```

## Completion Standard

- The patch is minimal and directly tied to the requested task.
- Relevant tests, lint, or type checks were run, or the reason they were not run is stated.
- PR descriptions include what changed, why it changed, tests run, and risk areas.
- P0/P1 review findings from Codex or auxiliary reviewers are addressed before requesting human merge.
