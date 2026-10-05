# Codebase map

A directory-by-directory guide. For each area: what it holds, the README that
governs it, and the rules that tests enforce there.

```
text-to-cad/
├── VERSION                  canonical release version (0.7.14). Never hand-edit other version strings
├── package.json             npm workspace root (core, ui, web, mcp, docs) and top-level scripts
├── package-lock.json        the ONLY lockfile
├── requirements-dev.txt     Python dev env: editable cadgen + zig + test extras
├── AGENTS.md                repo rules for AI agents (and humans): read first
├── CONTRIBUTING.md          setup, testing, CI graph, release process
├── README.md                end-user install instructions per agent app
├── SECURITY.md, LICENSE     MIT
│
├── .claude-plugin/          Claude Code plugin manifest + marketplace + icon
├── .codex-plugin/           Codex plugin manifest + logo
├── .cursor-plugin/          Cursor plugin manifest
├── gemini-extension.json    Gemini extension manifest (includes its MCP config)
├── claude.mcp.json          MCP server config the Claude plugin starts (cadgen mcp)
├── codex.mcp.json           ... for Codex
├── cursor.mcp.json          ... for Cursor Marketplace
├── skills.sh.json           grouping of skills on skills.sh
│
├── skills/                  THE PRODUCT: agent skills
├── packages/
│   ├── cadgen/              Python distribution published to PyPI
│   ├── core/                @text-to-cad/core: non-React 3D/CAD runtime
│   └── ui/                  @text-to-cad/ui: React FileViewer + renderers
├── apps/
│   ├── web/                 CAD Viewer client (served by `cadgen viewer`)
│   ├── mcp/                 CAD app page for agent hosts (served by `cadgen mcp`)
│   └── docs/                texttocad.dev + api.texttocad.dev (Next.js, Vercel)
├── models/                  sample/showcase CAD projects, manual use only
├── tests/                   root-owned test suites (Python, browser e2e, fixtures)
├── scripts/                 durable repo commands (bundle, test, release, install)
├── docs/                    migration guides + this onboarding folder
├── .github/                 CI workflows, setup-deps action, dependabot
└── .githooks/pre-commit     runs scripts/git-hooks/pre-commit (bundle --check)
```

---

## `skills/`

One folder per skill: `SKILL.md` (front-matter `name`/`description`, then
instructions), `references/*.md`, optional `scripts/`, `agents/openai.yaml`
(Codex listing metadata), `LICENSE`.

| Skill | Where the meat is |
|-------|-------------------|
| `cad` | `references/step-generation.md` (model contract), `build123d-modeling.md`, `positioning.md`, `kinematics.md`, `snapshot-review.md`, `repair-loop.md` |
| `dxf`, `engineering-drawing` | 2D outputs from build123d geometry |
| `urdf`, `srdf`, `sdf` | Robot descriptions. Validation logic is in `cadgen.{urdf,srdf,sdf}_*` |
| `step-parts`, `dfm`, `dfam-check`, `gcode`, `bambu-labs`, `sendcutsend` | cadgen-free skills with their own scripts/requirements |

Rules (enforced by `tests/python/global/test_skill_self_containment.py` and
`test_package_boundaries.py`):
- A skill never imports another skill, `skills/`, or the repo root, and never
  adds them to `sys.path`/`PYTHONPATH`/`NODE_PATH`.
- A cadgen skill runs cadgen only through the pinned `uvx ... cadgen==<VERSION>`
  command. It has no `requirements.txt` naming cadgen and no vendored copy.
- Skill scripts are thin entrypoints. Parsers and behaviour live in `cadgen.cli`.

## `packages/cadgen/` (Python, PyPI `cadgen`)

Governing docs: `README.md` (the 19 design laws, also the PyPI page),
`STORE.md` (store contract), `SNAPSHOTS.md` (snapshot timing/debug).

`src/cadgen/` highlights:

| Path | Role |
|------|------|
| `__init__.py`, `authoring.py`, `generation.py` | Public decorators (`step`, `dxf`, mesh) and the model-run pipeline |
| `build123d.py` / `.pyi` | `from cadgen import build123d as bd`: build123d re-export |
| `step.py`, `step_export*.py`, `step_scene.py`, `step_topology_artifact.py` | STEP writing, scene reading (`read_scene`), topology |
| `stl.py`, `threemf.py`, `glb.py` | Mesh exports |
| `dxf.py`, `eng_drawing.py`, `drawing_*.py`, `flatten.py` | 2D drawing and PDF engineering drawings |
| `urdf*.py`, `srdf*.py`, `sdf*.py` | Robot description authoring and validation |
| `assembly*.py`, `kinematics.py`, `instances.py`, `interference.py` | Assemblies, mates, joints, clash checks |
| `geometry.py`, `analysis.py`, `features.py`, `selector_types.py` | Measurement and selection helpers |
| `snapshot_core.py`, `snapshot_cli.py`, `snapshot_video.py`, `render.py` | Headless rendering via Playwright + `_runtime/browser` |
| `store/` | Content-addressed cache: paths, build records, meshes, gc |
| `daemon/` | Warm build daemon: server, client, worker pool, broker, memory admission |
| `coordination/` | Cross-process locks and paths |
| `cli/` | Every `cadgen <verb>`. The command table is in `cli/__init__.py` (`_COMMANDS`) |
| `viewer/` | CAD Viewer HTTP backend (`python -m cadgen.viewer`): catalog scanner, recents, file serving |
| `mcp/` | MCP stdio server: tools `cad_show`, `cad_open`, `cad_view`, `cad_screenshot`, ... |
| `analytics.py`, `updates.py`, `_internal/channel.py`, `_internal/api.py` | Opt-in analytics, daily version check, install channel |
| `_internal/` | Launch command, determinism shims, node runtime bridge, file tracer loader, etc. |
| `_runtime/` | **Built, gitignored.** `node/`, `browser/`, `viewer/`, `mcp/`, `native/` |

`native/filetrace.c` is the file tracer. zig cross-compiles it for every
platform into `_runtime/native`. It hooks the C library's `open`
(`CreateFile` on Windows) and logs every file a build reads or writes. The
store uses that log to decide whether a model is stale.

Laws to keep in mind: packaged docs must not reference repo paths ("ships
alone"). There are no backwards-compat shims. Byte-deterministic outputs. Nothing in
`cadgen.viewer` imports the CAD kernel at module scope. The Python floor is
declared only in `pyproject.toml` (`>=3.11`).

## `packages/core/` (`@text-to-cad/core`)

Governing docs: `README.md`, `docs/render-pipeline.md`,
`docs/resource-ownership.md`, `docs/tube-deformation.md`,
`docs/workspace-resources.md`.

Non-React TypeScript/JS: `src/lib/{step,glb,surf,urdf,render,viewer,drawing,
drawing2d,export,assembly,selectors}`, `src/client`, `src/prompt` (portable
prompt context), `bin/` and `scripts/`. It is built with esbuild via
`scripts/build/library.mjs`. It also produces the Node builders and the snapshot
browser bundle that cadgen ships. `bench/` holds benchmark helper units.

Rule: no React here. Everything depending on React goes in `packages/ui`.

## `packages/ui/` (`@text-to-cad/ui`)

Governing docs: `README.md`, `docs/settings-ui.md` (**binding** for any settings
control), `docs/viewer-host.md` (the host contract), `file-viewer.md`,
`renderers.md`, `cad-renderer.md`, `render-types.md`, `render-mode.md`, `lod.md`,
`view-updates.md`, `drawing.md`, `feature-detection.md`. AGENTS.md also lists
`storage.md`/`backend.md` here, but those live in `apps/web/docs/`.

`src/file-viewer/` (the FileViewer), `src/renderers/{step,mesh,dxf,glb,robot,
workspace}` plus `renderers/kit/` (camera, inspector, tools: draw, pose,
quick-edit, select, playbar; view settings; viewport), `src/host/` (injected
capabilities), `src/tab-store/`, `src/primitives/`, `src/styles/`.

Rules: platform-agnostic. Apps inject environmental effects through
capabilities and named slots. Browser specs are `*.browser.test.mjs`.

## `apps/web/` (CAD Viewer client)

Governing doc: `README.md` (launch contract, dev vs prod, behaviours that look
like bugs but are not). Also `docs/backend.md`, `shell.md`, `storage.md`.

Small app: `src/App.tsx`, `src/main.tsx`, `src/host/` (cad client, clipboard,
auto-reload, links), `src/persistence/`, `src/shared/viewerConfig.mjs`.
`vite.config.mjs` spawns the Python backend in dev. `.env.example` lists dev
variables. `scripts/` holds the dev-server helpers (directory root, fs allow,
lifetime) and their tests.

Rules: imports `@text-to-cad/core` and `@text-to-cad/ui` only by public exports
(`scripts/selfContained.test.mjs`). It never imports another app.

## `apps/mcp/` (CAD app for agent hosts)

Governing doc: `README.md`. `src/App.tsx`, `ModelView.tsx`, `src/host/`
(bridge to the MCP Apps host, capture, presentation, sync, tunnel). Builds to one
`index.html` that cadgen ships as `_runtime/mcp`. Presentation is **tabs**
(Codex) or **inline cards** (Claude Desktop and other MCP Apps hosts). Hosts
that render no MCP Apps get a viewer link instead.

## `apps/docs/` (texttocad.dev)

Governing doc: `README.md`. Next.js app router in `src/app/`. The homepage copy
is in `src/lib/content.ts` and must stay in sync with the root `README.md`
(enforced by `tests/python/global/test_plugin_manifests.py`). `/llms.txt`
renders the homepage as markdown for agents.

API (`src/app/v1/[...route]/route.ts` → `src/lib/api/handler.mjs`):
`GET /v1/versions`, `POST /v1/events`, `POST /v1/forget`, `GET /v1/prune`
(cron), `GET /v1/health`. Postgres store is in `postgres.mjs`, tables in `schema.sql`.
Any new analytics field also requires a privacy-policy change in the same PR.

`public/hero/` holds committed plain-file assets of the planetary gear model.
Refresh them with `apps/docs/scripts/sync-hero-step-assets.mjs`.

## `models/`

Governing doc: `models/README.md`, each project's `src/README.md` catalog.

| Folder | Content |
|--------|---------|
| `examples/` | Standalone parts, one script each, in `src/` |
| `assemblies/` | Assemblies, one `src/<assembly>/` group each |
| `drawings/` | 2D `@dxf` drawings |
| `f1`, `f14d`, `hypercar`, `moonwatch`, `motorbike`, `qdd_actuator`, `radial`, `w16`, `falcon_heavy`, `tendon_hand` | Showcase projects |
| `juno`, `lyra` | Robot description packages (URDF/SRDF) |
| `dfm-process-comparison` | Same tray designed for CNC/FDM/injection/sheet |
| `branding` | The logo as CAD |
| `tests/` | Manual edge-case models. **Not** CI input despite the name |

Each project has the shape `src/` (scripts, `src/lib/` helpers) plus format
folders (`STEP/`, `DXF/`, `STL/`, `3MF/`, `GLB/`). Outputs and `*.source.json`
sidecars are gitignored, so a fresh clone has no `STEP/` until you run the
scripts. Images and video under `models/` are gitignored.

## `tests/`

| Path | Covers |
|------|--------|
| `tests/python/packages/cadgen/` | Engine, daemon, CLI. `viewer/` is the viewer backend |
| `tests/python/skills/<skill>/` | Each skill's CLI behaviour |
| `tests/python/global/` | Repo-wide policy: boundaries, self-containment, plugin manifests/branch/zip, CI wiring, version pins |
| `tests/python/support/` | Shared helpers (temp dirs with Windows retries, `warm_daemon.py`) |
| `tests/browser/viewer-e2e.mjs` | Real-browser viewer gates (formats, camera) |
| `tests/fixtures/cad/` | Tiny test-owned fixtures |

JS unit tests live beside their sources (`*.test.mjs`, `*.test.ts`) and run
through each package's `scripts/run-tests.mjs`.

Rules: tests never read `models/`. Every test file is reached by a runner that
CI calls (`test_ci_workspace_selection.py`). Tests use no fixed sleeps. Flaky
tests are fixed or deleted, never retried.

## `scripts/`

Governing doc: `scripts/README.md` (full index).

| Folder | Key entries |
|--------|-------------|
| `bundle/` | `bundle.sh` (the one entry point), `cadgen-runtime.sh` (per-stage: `--node --browser --viewer --mcp --native`) |
| `test/` | `test.sh`, `test-js.sh`, `test-python.sh`, `test-global.sh`, `test-docs.sh`, `test-installed.sh`, `test-viewer-*.sh`, boundary checkers |
| `release/` | `check-version.sh`, `bump-version.sh`, `sync-version.mjs`, `check-wheel-contents.sh`, `plugin_zip.py`, `plugin_branch.py`, `publish-github-release.sh` |
| `install/` | `dev_install.py <host>` |
| `github-workflows/` | `check-builds.sh` (shipping contract), `deploy-vercel-app.sh` |
| `bench/` | Manual performance benchmarks. Output goes to `tmp/`, never committed |
| `brand/` | Logo generators |

Rule: only durable commands go here. One-off helpers go in `tmp/`.

## `.github/workflows/`

| Workflow | Trigger | Does |
|----------|---------|------|
| `test.yml` | Push/PR to `main`, manual | Path-conditional jobs: Version Check, cadgen (Linux/Windows), core-js, web, skills, codex, docs, packaging |
| `release-prepare.yml` | Manual | Bump `VERSION`, stamp pins, open and merge the release PR |
| `release-publish.yml` | Push to `main`/`build-test`, manual | Bundle, test, wheel, PyPI, docs deploy, tag, GitHub Release, `plugin` branch |
| `deploy-docs.yml` | Manual, called by publish | Vercel production deploy of `apps/docs` |

## Where to change what

| I want to... | Edit | Then run |
|--------------|------|----------|
| Change what the agent is told to do | `skills/<skill>/SKILL.md` or `references/` | `scripts/test/test-python.sh --select skills`, plus test in an agent via `dev_install.py` |
| Fix model generation / STEP export | `packages/cadgen/src/cadgen/` | `scripts/test/test-python.sh --select cadgen` |
| Add a CLI flag | The verb's function signature (many CLIs are generated from it) or `cli/<verb>.py` | Same as above |
| Change rendering / loaders | `packages/core/src/lib/...` | `npm run build:packages`, `scripts/test/test-js.sh --select core`, `bundle.sh` |
| Change viewer UI / tools / settings | `packages/ui/src/...`, following `docs/settings-ui.md` | `scripts/test/test-js.sh --select ui` / `web`, dev mode for manual check |
| Change the viewer backend | `packages/cadgen/src/cadgen/viewer/` | `scripts/test/test-python.sh --select viewer` |
| Change the in-chat app | `apps/mcp/src/`, `packages/cadgen/src/cadgen/mcp/` | `scripts/test/test-js.sh --select mcp`, `dev_install.py` |
| Change the website | `apps/docs/src/` (copy in `src/lib/content.ts`, kept in sync with root `README.md`) | `npm --prefix apps/docs run check` |
| Add a sample model | `models/<project>/src/` | `python models/<project>/src/<file>.py` |
