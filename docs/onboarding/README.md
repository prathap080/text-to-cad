# Onboarding: start here

This folder is a reading guide for people new to the **text-to-cad** repository.
It sits on top of the repo's own documentation and points into it; where the two
disagree, the repo's own files (`AGENTS.md`, `CONTRIBUTING.md`, each package's
`README.md`) are authoritative.

Version described: **0.7.14** (the `VERSION` file at the repo root).

## Read in this order

| # | File | What you get | Time |
|---|------|--------------|------|
| 1 | [PROJECT_OVERVIEW.md](PROJECT_OVERVIEW.md) | What the project is, who uses it, the moving parts and how a request flows through them | 10 min |
| 2 | [RUNBOOK.md](RUNBOOK.md) | Prerequisites, setup, every way to run it, environment variables, tests, troubleshooting | 20 min, then reference |
| 3 | [CODEBASE_MAP.md](CODEBASE_MAP.md) | Directory-by-directory map, where to change what, the rules each area enforces | Reference |

Then, depending on what you will work on:

| Working on | Read next |
|------------|-----------|
| The Python engine (`cadgen`) | `packages/cadgen/README.md` (the 19 design laws), then `packages/cadgen/STORE.md` |
| A skill | `skills/<name>/SKILL.md` and its `references/` |
| The CAD Viewer (browser app) | `apps/web/README.md`, then `packages/ui/README.md` |
| The in-chat CAD app (MCP) | `apps/mcp/README.md` |
| Shared 3D/runtime JS | `packages/core/README.md`, `packages/core/docs/` |
| The docs website / analytics API | `apps/docs/README.md` |
| Releases and CI | `CONTRIBUTING.md` (Releases, CI), `scripts/README.md` |

## Fastest path to seeing something work

On Windows with Python 3.11+ and Node 22 (both already on this machine):

```powershell
# 1. Python environment with the editable cadgen
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -r requirements-dev.txt

# 2. JS dependencies + build the packaged runtime (Node builders, viewer, tracer)
npm ci
& 'C:\Program Files\Git\bin\bash.exe' scripts/bundle/bundle.sh

# 3. Build a sample model and open it in the viewer
.\.venv\Scripts\python.exe models\examples\src\l_bracket.py
.\.venv\Scripts\python.exe -m cadgen.cli viewer
```

Details, alternatives and failure modes are in [RUNBOOK.md](RUNBOOK.md).

## Things that will surprise you

- **There is no web service to deploy.** The product is a set of agent *skills*
  plus a Python package (`cadgen`) that runs locally on the user's machine. The
  only hosted piece is the docs site and its small analytics/version API.
- **Models are Python scripts.** You run `python model.py`; a decorator
  (`@step`, `@dxf`, ...) writes the STEP/DXF file. There is no `cadgen generate`.
- **Built JS is never committed.** `packages/cadgen/src/cadgen/_runtime/` is
  gitignored. A fresh checkout cannot build a model until
  `scripts/bundle/bundle.sh` has run once.
- **Windows 11 Smart App Control blocks the CAD kernel.** If you see
  `ImportError: DLL load failed while importing OCP`, read the Troubleshooting
  section of the runbook before anything else.
- **This copy is a ZIP download, not a git clone.** Building and running work.
  Scripts that inspect git history (`check-builds.sh`, release scripts, the
  pre-commit hook) will not work until you `git clone` the repository instead.
