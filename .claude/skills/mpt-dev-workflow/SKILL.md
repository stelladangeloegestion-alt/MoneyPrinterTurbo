---
name: mpt-dev-workflow
description: Use when making code changes to the MoneyPrinterTurbo codebase (app/, cli.py, main.py, webui/, test/) — how to set up the environment with uv, run the same compile/lint/test/coverage checks as CI before pushing, and how this repo's two "skill" concepts differ. Trigger on requests to fix a bug, add a feature, add or update tests, or review a diff in this repo.
---

# Working in MoneyPrinterTurbo

MoneyPrinterTurbo is a FastAPI + Streamlit app (`app/`, `webui/`) with a CLI
(`cli.py`, `main.py`) for generating short videos from prompts/scripts. Python
3.11+ is required; dependencies are pinned via `uv.lock`. Always use `uv` —
never plain `pip` or a hand-managed venv (`pyproject.toml` sets
`[tool.uv] package = false`; this is an application, not an installable
package).

## Two unrelated "skills" in this repo — don't confuse them

- **`docs/skill/SKILL.md`** is a *distributable, end-user-facing* Claude
  Skill (`moneyprinterturbo-video`) published for AI agents to install and
  run MoneyPrinterTurbo end-to-end and hand back a finished MP4. It ships
  with `docs/skill/mpt_agent.py`. Treat it as product documentation/code:
  only touch it when the task is specifically about that generation
  workflow (installation, credential handling, exit codes, defaults).
- **`.claude/skills/`** (this directory) holds *repo-development* skills —
  instructions for Claude Code sessions that are editing this codebase, like
  the one you're reading now. Never merge the two or copy one's conventions
  into the other; they serve different audiences.

## Environment setup

```bash
uv sync --frozen --python 3.11
```

Use `--python 3.13` instead if specifically validating the other CI-tested
version. `--frozen` matches CI: it fails rather than silently updating
`uv.lock` if dependencies drift.

## Before pushing, reproduce CI locally (`.github/workflows/ci.yml`)

Run these from the repo root, in order — they are the exact commands CI runs:

```bash
uv run --no-sync python -m compileall app cli.py main.py webui test
uv run --no-sync ruff check app cli.py main.py webui test
uv run --no-sync python -X utf8 -m coverage run -m pytest -q test
uv run --no-sync python -m coverage report
```

- Coverage is branch coverage over `app`, `cli`, `webui`, `main`, and
  `docs/skill` (see `[tool.coverage.run]`/`[tool.coverage.report]` in
  `pyproject.toml`) and CI fails under 70% (`fail_under = 70`). If you add
  code, add or extend tests in the same change rather than letting coverage
  slide.
- Redis-backed tests (`test_state.py`, `test_task.py`) auto-skip unless
  `MPT_TEST_REDIS_HOST` is set — you don't need a local Redis for a normal
  edit. To exercise them, run a local Redis and export
  `MPT_TEST_REDIS_HOST=127.0.0.1 MPT_TEST_REDIS_PORT=6379 MPT_TEST_REDIS_DB=15`
  (matches CI's redis service container).
- Live-provider tests (real TTS/LLM calls) are skipped by default; only set
  `MPT_RUN_INTEGRATION_TESTS=1` with real credentials if the task specifically
  requires exercising an external provider.
- There is also a Windows-only smoke job that runs a fixed subset of
  `test/services/` — if you touch config, state, task, task_manager,
  upload_post, controller_video, or webui_task, check that subset stays
  green; it's listed explicitly in `ci.yml`.

To run a single file/class/test during iteration:

```bash
uv run --no-sync python -X utf8 -m pytest -q test/services/test_video.py
uv run --no-sync python -X utf8 -m pytest -q test/services/test_video.py::TestVideoService::test_preprocess_video
```

## Test conventions (`test/README.md`)

- One file per domain, named `test_<domain>.py`, under `test/` or
  `test/services/`.
- Split broad controller suites into `test_controller_<domain>.py`.
- Either plain pytest functions or `unittest.TestCase` are fine — pytest
  collects both, and CI relies on that.
- Test resource files go in `test/resources/`.

## Secrets

Never print `config.toml`'s contents, API keys, or other credential-bearing
config in logs, test output, or commit messages — this mirrors the rule the
end-user skill (`docs/skill/SKILL.md`) already enforces for generated runs.
