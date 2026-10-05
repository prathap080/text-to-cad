# Runbook

How to set up, run, test and debug text-to-cad from a source checkout. It also
covers the environment variables the code reads.

Commands are given for **Windows (PowerShell)** first, since that is this
machine, then for **Linux/macOS/WSL**. Run everything from the repository root
unless a step says otherwise.

- [1. Prerequisites](#1-prerequisites)
- [2. One-time setup](#2-one-time-setup)
- [3. Ways to run it](#3-ways-to-run-it)
- [4. Environment variables](#4-environment-variables)
- [5. Tests and checks](#5-tests-and-checks)
- [6. Where things live on disk](#6-where-things-live-on-disk)
- [7. Troubleshooting](#7-troubleshooting)
- [8. Releasing and deploying (maintainers)](#8-releasing-and-deploying-maintainers)

---

## 1. Prerequisites

| Tool | Version | Needed for | Check |
|------|---------|-----------|-------|
| Python | **≥ 3.11** (CI uses 3.12) | Everything in cadgen | `python --version` |
| Node.js | **22** (≥ 20 works for cadgen's Node builders) | Bundling the runtime, viewer, docs, JS tests | `node --version` |
| npm | Bundled with Node | Workspace install | `npm --version` |
| Git for Windows (Git Bash) | Any recent | Running the repo's `.sh` scripts on Windows | `& 'C:\Program Files\Git\bin\bash.exe' --version` |
| uv / uvx | Latest | Only to run cadgen the way end users do (plugin, skills) | `uv --version` |
| git | Any | Only for scripts that inspect history (see note below) | `git --version` |
| ffmpeg | Optional | `snapshot --video` | `ffmpeg -version` |

Installed toolchain on this machine at the time of writing: Python 3.12.10 (3.11
also present), Node v22.17.0, npm 10.9.2, git 2.44. **uv is not installed.** You
only need it for section 3.5.

Install uv if you need it:

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

> **This copy is a ZIP download, not a git clone.** Setup, builds, the viewer and
> the Python/JS test suites all work. `scripts/github-workflows/check-builds.sh`,
> the release scripts and the pre-commit hook call `git ls-files`/`git grep`
> and will fail. To contribute upstream, clone instead:
> `git clone https://github.com/earthtojake/text-to-cad.git`.

Disk and network: the first install downloads build123d, OCP (~100+ MB), the
npm workspace, and on first snapshot a headless Chromium. Plan for a few GB.

---

## 2. One-time setup

### 2.1 Python environment (required)

`requirements-dev.txt` installs `packages/cadgen` **editable**, plus zig (to
compile the native file tracer) and the test extras (pytest, trimesh, scipy,
rtree, networkx, lxml).

Windows (PowerShell):

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -r requirements-dev.txt
```

Linux / macOS / WSL:

```bash
python3.12 -m venv .venv
./.venv/bin/python -m pip install --upgrade pip
./.venv/bin/python -m pip install -r requirements-dev.txt
```

On WSL, install inside the distribution. Do not share a `.venv` or
`node_modules` across the Windows/WSL boundary.

Re-run the `pip install -r requirements-dev.txt` line after pulling new code.
It refreshes the editable install's metadata.

Verify the kernel loads:

```powershell
.\.venv\Scripts\python.exe -c "import build123d, OCP; print('kernel ok')"
.\.venv\Scripts\python.exe -m cadgen.cli doctor skills/cad
```

If this fails with `DLL load failed while importing OCP`, go to
[Troubleshooting](#smart-app-control-blocks-ocp-windows-11).

### 2.2 JavaScript workspace

There is one root `package-lock.json` for five workspaces. Install all of them:

```powershell
npm ci
```

Or only what your task needs (faster):

```powershell
# Python engine / shared core only
npm ci --workspace @text-to-cad/core

# Viewer and shared UI
npm ci --workspace @text-to-cad/core --workspace @text-to-cad/ui --workspace @text-to-cad/web

# Docs site
npm ci --workspace @text-to-cad/core --workspace cad-skills-docs
```

Do not create nested lockfiles or run `npm install` inside a sub-package.

### 2.3 Build the packaged runtime (required before building models)

`packages/cadgen/src/cadgen/_runtime/` is gitignored and starts empty. It holds
the Node builders, the snapshot browser bundle, the built viewer, the built MCP
page and the native file tracer. A fresh checkout **cannot build a model** until
this has run.

Windows (scripts are bash, run through Git Bash; they find `.venv\Scripts\python.exe` on their own):

```powershell
$gitBash = 'C:\Program Files\Git\bin\bash.exe'
& $gitBash scripts/bundle/bundle.sh          # build everything
& $gitBash scripts/bundle/bundle.sh --check  # build and assert completeness
& $gitBash scripts/bundle/bundle.sh --clean  # wipe _runtime first, then build
```

Linux / macOS / WSL:

```bash
scripts/bundle/bundle.sh
```

Re-run after you edit anything in `packages/core`, `packages/ui`, `apps/web` or
`apps/mcp` that cadgen consumes. The output is not committed.

### 2.4 Browsers for snapshots and browser tests (optional)

Snapshots fetch Playwright's headless shell on first use automatically. To
pre-install it, plus the npm Chromium that UI/web browser tests use:

```powershell
.\.venv\Scripts\python.exe -m playwright install --only-shell chromium
npx --no-install playwright install chromium
```

On Linux add `--with-deps`.

---

## 3. Ways to run it

| # | What | Use when |
|---|------|----------|
| 3.1 | Build a model script | You want a STEP/DXF/mesh file |
| 3.2 | `cadgen` CLI | Snapshots, exports, validation, store/daemon admin |
| 3.3 | CAD Viewer (production mode) | Look at models in the browser |
| 3.4 | CAD Viewer (dev mode, HMR) | Develop the viewer UI |
| 3.5 | MCP server / agent app | Test the plugin the way users get it |
| 3.6 | Docs site | Work on texttocad.dev |

On Windows, replace `python` below with `.\.venv\Scripts\python.exe`, or
activate the venv first with `.\.venv\Scripts\Activate.ps1`. On POSIX use
`./.venv/bin/python` or `source .venv/bin/activate`.

### 3.1 Build a model

A model is a Python script whose decorated function writes the artifact. Paths
in `out=` are relative to the script.

```powershell
python models\examples\src\l_bracket.py          # -> models\examples\STEP\l_bracket.step
python models\drawings\src\gasket_plate.py       # -> models\drawings\DXF\gasket_plate.dxf
```

Build a whole sample project (unchanged models are no-ops):

```bash
cd models/examples
ls src/*.py | xargs -n1 -P4 python
```

Each project's `src/README.md` lists which script builds which artifact. For
your own scratch models, put a script in `models/examples/src/` (parts),
`models/assemblies/src/<name>/` (assemblies) or `models/drawings/src/` (DXF).

### 3.2 The `cadgen` CLI

Run as `python -m cadgen.cli <verb>` (always works) or `cadgen <verb>` (when the
venv's `Scripts`/`bin` is on PATH). `--help` works on every verb.

| Command | Does |
|---------|------|
| `cadgen doctor skills/cad` | Prints installed cadgen, checks the skill's pin and that the kernel loads |
| `cadgen step snapshot IN.step out.png [--display render] [--camera front]` | Render a STEP to an image (`--mode section --section XZ:12.5` for cuts, more in `--help`) |
| `cadgen step build IN.step OUT.step` | Re-emit a STEP, optionally adding kinematics, materials, animation |
| `cadgen stl build IN.step [OUT.stl]` | Tessellate to STL. Also `3mf build`, `glb build` |
| `cadgen stl snapshot IN.stl` | Render a mesh. Also `3mf`, `glb`, `dxf`, `urdf`, `sdf` snapshot |
| `cadgen snapshot FILE` | Render any supported input |
| `cadgen urdf validate robot.urdf` | Validate. Also `sdf validate`, `srdf validate` |
| `cadgen store info` / `why <model>` / `forget <target>` / `gc` | Inspect and clean the build cache |
| `cadgen daemon status` | Show the warm build daemon's workers |
| `cadgen viewer` / `cadgen viewer stop` | Start/reuse or stop the CAD Viewer |
| `cadgen mcp` | Serve CAD to an agent host over MCP stdio |
| `cadgen analytics status\|on\|off` | Anonymous usage analytics opt-in |

Check exact flags with `--help` before scripting against them.

### 3.3 CAD Viewer: production mode

```powershell
python -m cadgen.cli viewer                       # http://127.0.0.1:3245, stays in foreground
python -m cadgen.cli viewer --port 4000
python -m cadgen.cli viewer --detach              # background, logs to viewer.log in the state dir
python -m cadgen.cli viewer --new                 # random free port, never reused
python -m cadgen.cli viewer stop                  # stop the one on 3245
```

Open a file with `http://127.0.0.1:3245/?file=C:/abs/path/part.step`.

Behaviour to know:

- If a viewer with the same code already runs on that port, the launch
  **reuses** it and exits. If one with different code runs, it is asked to
  exit and replaced. If another program holds the port, it refuses.
- In a checkout, a built `apps/web/dist` takes priority over the packaged
  copy. To serve this checkout's web build explicitly:
  ```powershell
  npm run build:web
  $env:CADGEN_VIEWER_DIST = "$PWD\apps\web\dist"
  python -m cadgen.cli viewer --json
  ```
- A running viewer that detects its code changed returns "restart required".
  Stop it and start again.
- Never stop a viewer you did not start (an agent app may own it).

### 3.4 CAD Viewer: dev mode (Vite + HMR)

Build the shared packages first, then run Vite **from a folder of models**
(outside `apps/web`). Relative `?file=` paths resolve against that folder.

Windows:

```powershell
npm run build:packages
cd C:\path\to\models-folder
$env:VIEWER_PYTHON = "C:\Users\saipr\Downloads\text-to-cad-main\text-to-cad-main\.venv\Scripts\python.exe"
npm --prefix C:\Users\saipr\Downloads\text-to-cad-main\text-to-cad-main\apps\web run dev -- --host 127.0.0.1
# open http://127.0.0.1:5173/?file=STEP/l_bracket.step
```

POSIX:

```bash
npm run build:packages
cd <a folder of models>
VIEWER_PYTHON=<checkout>/.venv/bin/python \
  npm --prefix <checkout>/apps/web run dev -- --host 127.0.0.1
```

Vite spawns its own Python backend (`python -m cadgen.viewer --new --api-only`)
and proxies `/__cad` and `/__tess_cache` to it. Port 5173 must be free, or pass
`--port`. Shortcut from the repo root: `npm run dev:web`. It builds the packages
and starts Vite. Relative `?file=` paths then resolve against the repo root,
which still needs `VIEWER_PYTHON` set.

### 3.5 MCP server and agent apps

**As end users get it** (needs uv). This is exactly what the plugin configs run:

```powershell
uvx --no-config --managed-python --python 3.13 --from cadgen==0.7.14 cadgen mcp
```

It speaks MCP over stdio, so on its own it will just wait for input. It is meant
to be launched by an agent app.

**From this checkout** (contributors). Install the checkout as a dev plugin
into one agent app. The server and skills then use this checkout's `.venv`:

```powershell
python scripts\install\dev_install.py claude          # also: codex, cursor, grok, gemini, claude-desktop
python scripts\install\dev_install.py claude --wheel  # build a wheel and run that instead
python scripts\install\dev_install.py claude --uninstall
```

Rules:
- Only one copy of the plugin per app. Uninstall the published plugin first.
  The script refuses otherwise.
- Restart the agent app after Python changes. A running server keeps its
  interpreter.
- Check the server connected: `claude mcp list` (Claude Code), `gemini mcp list`.

Server logs per host are listed in `CONTRIBUTING.md`, under "Test In Agent Apps".

**Install the published plugin** (not from source). See the root `README.md`.
In short, for Claude Code:

```bash
claude plugin marketplace add earthtojake/text-to-cad
claude plugin install text-to-cad@earthtojake
```

### 3.6 Docs site (`apps/docs`)

```powershell
npm ci --workspace @text-to-cad/core --workspace cad-skills-docs
npm run dev:docs                      # builds core, then `next dev` on http://localhost:3000
npm --prefix apps/docs run check      # CI gate: assets, API tests, lint, build, icon verify
```

The site renders statically without any env vars. Only the `/v1/events`,
`/v1/forget`, `/v1/prune` and `/v1/health` routes need `DATABASE_URL` and
`CRON_SECRET`. See section 4.4.

---

## 4. Environment variables

**None are required for local development.** Every variable below has a working
default. They exist to override paths, tune performance, control privacy, or
wire up the hosted docs/API.

### 4.1 Privacy and network

| Variable | Default | Effect |
|----------|---------|--------|
| `DO_NOT_TRACK` | unset | `1` forces analytics off (strongest setting) |
| `CADGEN_ANALYTICS` | unset (asks the user once) | `0` off, `1` on. Overrides the saved choice |
| `CADGEN_UPDATE_CHECK` | on | `0` disables the once-a-day fetch of `api.texttocad.dev/v1/versions` |
| `CI` | unset | When set, update checks never run |
| `CADGEN_API_URL` | `https://api.texttocad.dev` | Point analytics and version checks at another `http(s)` receiver (local testing) |

### 4.2 Install identity (set by plugin configs, not by you)

| Variable | Typical values | Effect |
|----------|----------------|--------|
| `CADGEN_INSTALL_CHANNEL` | `claude-github`, `codex-github`, `gemini-github`, `claude-desktop`, `cursor-marketplace`, `claude-directory`, `openai-directory`, `dev`, `unknown` | Where this install came from. Reported in analytics |
| `CADGEN_AUTO_UPDATED` | `1` or unset | Something else (a store, Gemini) updates this copy, so never show the update button |

To see the update button during development, run with
`CADGEN_INSTALL_CHANNEL=claude-github` and put a `versions.json` naming a newer
`latest` in the state directory.

### 4.3 cadgen paths, runtime and performance

| Variable | Default | Effect |
|----------|---------|--------|
| `CADGEN_CACHE_DIR` | Windows `%LOCALAPPDATA%\cadgen`. POSIX `$XDG_CACHE_HOME/cadgen` or `~/.cache/cadgen` | Location of the build store (cache) |
| `CADGEN_STORE_MAX` | `20G` | Store size cap the daemon evicts down to when idle. `0` means no cap |
| `CADGEN_STATE_DIR` | Windows `%LOCALAPPDATA%\cadgen-state`. macOS `~/Library/Application Support/cadgen`. Linux `$XDG_STATE_HOME/cadgen` or `~/.local/state/cadgen` | Settings, recents, `versions.json`, `viewer.log` |
| `CADGEN_DAEMON` | on | `0` keeps builds cold, with no warm daemon. Tests use this |
| `CADGEN_DAEMON_IDLE_TIMEOUT` | `3600` seconds | Idle time before the daemon exits |
| `CADGEN_DAEMON_SOCKET` | Derived per installation | Override the daemon address (POSIX path or Windows pipe name) |
| `CADGEN_DAEMON_STATE_DIR` | `<tempdir>/cadgen-daemon` | Daemon bookkeeping dir. Tests isolate daemons with it |
| `CADGEN_JOBS` | CPU count | Concurrent build jobs |
| `CADGEN_COMPONENT_WORKERS` | Auto | Parallel component workers. Set `1` on network drives that hit `WinError 32` |
| `CADGEN_MEMORY_MB` | 70 % of RAM | Daemon memory budget for admitting builds. `0` disables admission control |
| `CADGEN_COMPONENT_MEMORY_MB` | Built-in | Per-component memory reservation |
| `CADGEN_NODE` | `node` on PATH | Node.js binary the Node builders run with (needs ≥ 20) |
| `CADGEN_FFMPEG` | `ffmpeg` on PATH | ffmpeg for `snapshot --video` |
| `CADGEN_MESH_CACHE` | on | `0` disables the tessellation cache used by snapshots and the viewer |
| `CADGEN_VERIFY_READBACK` | off | Verify each built artifact by reading it back (debugging, STORE.md §10) |
| `CADGEN_DETERMINISM` | on | `0` disables the determinism shims over build123d (debugging only) |
| `CADGEN_FONT_GUARD` | on | `0` disables the font-scan guard |
| `CADGEN_VIEWER_DIST` | Checkout `apps/web/dist`, else packaged | Built viewer client to serve (same as `cadgen viewer --dist`) |
| `CADGEN_NODE_BUILDERS_DIR` | Checkout build, else packaged | Directory of the esbuilt Node builders |
| `CADGEN_BROWSER_RUNTIME_DIR` | Packaged | Snapshot browser bundle directory |
| `CADGEN_MCP_PRESENTATION` | Auto-detected | `inline` or `text` forces how `cadgen mcp` presents to a host that does not advertise MCP Apps |
| `CADGEN_MCP_APP_DIR` | Checkout build, else packaged | Built MCP app page to serve. `dev_install.py` sets it |
| `CADGEN_PYTHON` | The checkout `.venv` | Used only by `dev_install.py`: the interpreter the dev plugin's server runs |

Internal variables you should not set by hand include `CADGEN_DAEMON_CHILD`,
`CADGEN_ROOT_ID`, `CADGEN_JOB_ID`, `CADGEN_EVENTS`, `CADGEN_BROKER*`.
cadgen sets them on its own child processes.

### 4.4 CAD Viewer dev server (`apps/web`, read by Vite)

Template: `apps/web/.env.example`.

| Variable | Default | Effect |
|----------|---------|--------|
| `VIEWER_PYTHON` | `python3` | Interpreter for the spawned backend. Point it at `.venv`. Must be ≥ 3.11. STEP import needs cadgen installed |
| `VIEWER_BACKEND_URL` | unset | Attach to a backend you started yourself (e.g. `http://127.0.0.1:3400`) instead of spawning one |
| `VIEWER_ALLOWED_HOSTS` | Vite default | Extra hostnames Vite accepts, comma-separated |
| `VIEWER_SERVER_LIFETIME_MS` | unset (no limit) | Optional lifetime of the dev server, in ms |
| `VIEWER_DEFAULT_FILE` | unset | File opened by default |
| `VIEWER_GITHUB_URL`, `VIEWER_DISCORD_URL` | Upstream links | Override links shown in the UI |

> On Windows the `.env.example` sample `./.venv/bin/python` must be
> `.\.venv\Scripts\python.exe` (or an absolute path).

### 4.5 Docs site / `api.texttocad.dev` (`apps/docs`, production only)

Set in the Vercel dashboard as **Sensitive** production env vars. Nothing in
GitHub holds them. Not needed for `next dev` or `npm run check`.

| Variable | Required for | Notes |
|----------|-------------|-------|
| `DATABASE_URL` | `/v1/events`, `/v1/forget`, `/v1/prune`, `/v1/health` | Pooled Postgres connection string (Neon today). Run `apps/docs/src/lib/api/schema.sql` on the database before deploying a schema change |
| `CRON_SECRET` | `/v1/prune` (daily cron at 04:17 UTC, from `vercel.json`) | Any long random string. Vercel sends it as `Authorization: Bearer …` |
| `NEXT_PUBLIC_SITE_URL` | Optional | Overrides the canonical site URL |

`GET /v1/health` returns `503` naming whichever of `DATABASE_URL`/`CRON_SECRET`
is missing. `Deploy Docs` checks it.

### 4.6 Test runners

| Variable | Default | Effect |
|----------|---------|--------|
| `PYTHON_BIN` | `.venv` interpreter, else `python3` | Interpreter the `scripts/test/*.sh` runners use |
| `CADGEN_TEST_JOBS` | CPU count (CI: 4) | Parallel Python test files |
| `UI_BROWSER_TEST_CONCURRENCY` | 4 (CI: 1) | Parallel UI browser spec files |
| `CAD_TEST_SWIFTSHADER` | unset | `1` uses software WebGL on macOS to reproduce CI-only failures |
| `VIEWER_TEST_DIAGNOSTICS_DIR` | unset (set in CI) | Where viewer browser tests write renderer-state JSON on failure |

### 4.7 Release / CI secrets (GitHub, maintainers only)

The workflows in `.github/workflows/` use repository secrets for PyPI upload,
Vercel deploy and the release PAT. Their exact names are in those YAML files.
They are never needed locally.

---

## 5. Tests and checks

Run the smallest check that covers your change. On Windows, prefix `.sh`
scripts with `& 'C:\Program Files\Git\bin\bash.exe'`.

| Scope | Command |
|-------|---------|
| Everything (JS, then Python, then policy) | `scripts/test/test.sh` |
| JS suites | `scripts/test/test-js.sh --select core\|ui\|web\|mcp\|all` (or `npm test`) |
| Python: cadgen package incl. viewer backend | `scripts/test/test-python.sh --select cadgen` |
| Python: viewer backend only (~11 s) | `scripts/test/test-python.sh --select viewer` |
| Python: all skill suites | `scripts/test/test-python.sh --select skills` |
| Which Python tests are slow | `scripts/test/test-python.sh --print-weights` |
| Repo policy gates | `scripts/test/test-global.sh` |
| Docs site | `npm --prefix apps/docs run check` |
| Viewer in a real browser | `scripts/test/test-viewer-browser.sh [--only camera]` |
| Runtime builds and is complete | `scripts/bundle/bundle.sh --check` |
| JS dependency boundaries | `npm run check:boundaries` |
| Version pins consistent | `scripts/release/check-version.sh` |
| One Python test file | `.\.venv\Scripts\python.exe -m unittest tests/python/skills/urdf/test_cli.py` |

The Python runners build any missing runtime stages they need. Tests are
self-contained by policy: they never touch `models/`.

---

## 6. Where things live on disk

| What | Windows | Linux | macOS |
|------|---------|-------|-------|
| Build store (cache) | `%LOCALAPPDATA%\cadgen` | `~/.cache/cadgen` | `~/.cache/cadgen` |
| State (settings, recents, `viewer.log`, `versions.json`) | `%LOCALAPPDATA%\cadgen-state` | `~/.local/state/cadgen` | `~/Library/Application Support/cadgen` |
| Daemon state | `%TEMP%\cadgen-daemon` | `/tmp/cadgen-daemon` | `$TMPDIR/cadgen-daemon` |
| Packaged runtime (built) | `packages\cadgen\src\cadgen\_runtime\` | same | same |
| Viewer build | `apps\web\dist\` | same | same |
| Dev-install assembly | `tmp\<host>-dev\` | same | same |

The store holds only derived data. Deleting it is always safe and only costs
rebuild time. Prefer `cadgen store gc`.

---

## 7. Troubleshooting

### Smart App Control blocks OCP (Windows 11)

Symptom: `ImportError: DLL load failed while importing OCP` on any `cadgen`
command or `import build123d`. Event Viewer shows Event ID 3077 under
CodeIntegrity › Operational. `cadgen doctor` names it.

Cause: the OCP wheel ships an unsigned native module, and Smart App Control
blocks unsigned code. There is no per-app exception.

Fix, pick one:
1. Turn Smart App Control off: Settings › Privacy & security › Windows Security ›
   App & browser control › Smart App Control settings. Once off, it can only
   be turned back on by reinstalling Windows.
2. Work inside **WSL** (Ubuntu). Create a fresh `.venv` and `node_modules` there.

### "Node builder is missing" / "runtime … not found"

`_runtime/` was never built, or is stale. Run `scripts/bundle/bundle.sh`. If Node
is not on PATH, set `CADGEN_NODE` to the full path of `node.exe`.

### The viewer shows old UI or answers "restart required"

The viewer's code or the bundled client changed under a running instance. Run
`cadgen viewer stop`, then `cadgen viewer`. After editing `apps/web`, rebuild
with `npm run build:web` or use dev mode (3.4).

### Port 3245 or 5173 in use

`cadgen viewer --port N` or `--new`. Vite: `-- --port N`. The viewer refuses a port
held by another program or another user's viewer.

### `WinError 32` during a parallel build (network drives, antivirus)

Set `CADGEN_COMPONENT_WORKERS=1`. Working from a local disk also helps.

### First snapshot is slow or fails offline

The first snapshot downloads Playwright's headless Chromium. Pre-install it
with `python -m playwright install --only-shell chromium` while online.

### Builds feel slow every time

Check the warm daemon: `cadgen daemon status`. Make sure `CADGEN_DAEMON` is not
`0`. Ask why a model rebuilt: `cadgen store why <model.py>`. That also takes
`script.py::function` or a generated `.step`.

### Viewer dev server: "Python too old" / cannot import cadgen

`VIEWER_PYTHON` defaults to `python3`, which is often missing or old on Windows.
Point it at `.venv\Scripts\python.exe`.

### Scripts fail with `git: not a git repository`

This copy is a ZIP extract. See section 1. Clone the repository for git-aware
scripts.

### Two copies of every skill in the agent

The published plugin and a dev install, or loose skills, are both loaded. Keep
one: `python scripts\install\dev_install.py <host> --uninstall`, or remove the
published plugin.

---

## 8. Releasing and deploying (maintainers)

Do not bump `VERSION` in normal PRs. CI rejects it outside `release/*` branches.

| Task | Command |
|------|---------|
| Cut a release (always name the bump) | `gh workflow run release-prepare.yml --ref main -f bump=patch` |
| Resume or republish the current head | `gh workflow run release-publish.yml --ref main` |
| Rehearse without publishing | `git push origin main:build-test` then `gh workflow run release-prepare.yml --ref main -f bump=patch -f target=build-test` |
| Redeploy docs only | `gh workflow run deploy-docs.yml -f ref=main` |

`Publish Release` bundles, tests and builds the wheel, then uploads to PyPI,
deploys docs, tags `v<VERSION>`, creates the GitHub Release with the wheel,
sdist and OpenAI plugin ZIP, and updates the `plugin` branch. Full detail is in
`CONTRIBUTING.md` › Releases.
