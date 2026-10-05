# Project overview

## What text-to-cad is

text-to-cad gives an AI coding agent (Claude Code, Codex, Cursor, Gemini, Grok,
and others) the ability to **design physical parts**. The agent writes a short
Python script; running it produces real CAD files: STEP for engineering, STL,
3MF or GLB for printing and display, DXF for 2D cutting, URDF/SRDF/SDF for robots.

On top of generation it adds:

- a **viewer** to orbit, select and inspect models (in the browser, or as cards
  and tabs inside the agent's own app),
- **design-for-manufacturing checks** (sheet metal, CNC, injection molding, 3D
  printing),
- **engineering drawings** as PDF,
- hand-offs to **fabrication services** (SendCutSend, Bambu Lab printers,
  OrcaSlicer G-code).

It is MIT licensed, published on GitHub as `earthtojake/text-to-cad` and on
PyPI as `cadgen`. Documentation site: <https://www.texttocad.dev>.

## Who runs what

There are two audiences, and they run different things.

| Audience | What they install | What runs on their machine |
|----------|-------------------|----------------------------|
| **End users** (people using an agent) | The plugin or skills, via their agent app (`claude plugin install ...`, `npx skills add ...`) | `uvx ... cadgen==0.7.14 cadgen mcp` started by the agent app, plus `cadgen` commands the skills run |
| **Contributors** (you, working in this repo) | A Python `.venv` with the editable `cadgen`, and the npm workspace | The same `cadgen` commands, but from this checkout's source |

Nothing here is a hosted backend. The single hosted piece is the docs site
(Next.js on Vercel), which also answers `api.texttocad.dev` for the daily
version check and opt-in anonymous analytics.

## The building blocks

```
                   ┌──────────────────────────────────────────────┐
  Agent app        │  skills/*/SKILL.md   (instructions the agent  │
  (Claude, Codex…) │                       reads and follows)      │
                   └───────────────┬──────────────────────────────┘
                                   │ runs commands / MCP tools
                   ┌───────────────▼──────────────────────────────┐
                   │              cadgen  (Python, PyPI)          │
                   │  packages/cadgen/src/cadgen                  │
                   │                                              │
                   │  @step/@dxf decorators ─► build123d / OCP     │
                   │  (model scripts)           (CAD kernel)      │
                   │        │                                     │
                   │        ▼                                     │
                   │  store (cache) ◄── warm build daemon         │
                   │        │                                     │
                   │  cli/  (step build, snapshot, validate …)    │
                   │  viewer/ (HTTP server, port 3245)            │
                   │  mcp/    (stdio MCP server for agent apps)   │
                   │                                              │
                   │  _runtime/  (BUILT, not committed)           │
                   │    node/    Node builders (mesh export …)    │
                   │    browser/ headless snapshot bundle         │
                   │    viewer/  built apps/web                   │
                   │    mcp/     built apps/mcp                   │
                   │    native/  file tracer (zig-compiled C)     │
                   └───────────────▲──────────────────────────────┘
                                   │ built from
       ┌───────────────────────────┼───────────────────────────────┐
       │  packages/core  (3D runtime, no React)                    │
       │  packages/ui    (FileViewer, renderers, React)            │
       │  apps/web       (CAD Viewer client)                       │
       │  apps/mcp       (CAD app page shown inside agent hosts)   │
       └───────────────────────────────────────────────────────────┘

       apps/docs  (Next.js site + api.texttocad.dev) — separate, hosted on Vercel
```

### 1. Skills (`skills/`) — the product

Twelve skills. Each is a folder with a `SKILL.md` the agent reads, plus
`references/` docs and sometimes thin scripts.

| Skill | Purpose | Needs cadgen? |
|-------|---------|---------------|
| `cad` | Create/edit parametric 3D models, export STEP/STL/3MF/GLB, inspect, view | yes |
| `dxf` | 2D DXF drawings (profiles, gaskets, cut layouts) | yes |
| `engineering-drawing` | Dimensioned PDF drawings from a part | yes |
| `urdf` / `srdf` / `sdf` | Robot descriptions for ROS, MoveIt2, Gazebo | yes |
| `step-parts` | Find off-the-shelf STEP parts (screws, bearings, motors) | no |
| `dfm` | Manufacturability review for sheet metal, CNC, molding | no |
| `dfam-check` | 3D-print printability of a mesh | no |
| `gcode` | Slice with OrcaSlicer | no |
| `bambu-labs` | Send prints to Bambu Lab printers | no |
| `sendcutsend` | Pre-upload checks for SendCutSend | no |

Every skill that needs cadgen runs it through one pinned launch command:

```
uvx --no-config --managed-python --python 3.13 --from cadgen==0.7.14 cadgen ...
```

Because every caller uses the same requirement string, they all share one
installation and one warm daemon.

### 2. `cadgen` (`packages/cadgen/`) — the engine

A Python package (Python ≥ 3.11) built on **build123d** and **OCP** (the Python
binding of the OpenCascade kernel). It holds:

- **Authoring API**: decorators (`step`, `dxf`, mesh decorators) and a
  re-export of build123d as `cadgen.build123d`. A model is a plain script:

  ```python
  from cadgen import build123d as bd
  from cadgen import step

  @step(out="../STEP/bracket.step")
  def bracket():
      body = bd.Box(40, 20, 6)
      body.label = "bracket"
      return body

  if __name__ == "__main__":
      bracket()
  ```

- **The store**: a content-addressed cache of derived results (tessellations,
  topology, component packages) under the user cache directory. Rebuilds of
  unchanged models are no-ops. Contract: `packages/cadgen/STORE.md`.
- **The warm build daemon**: a background process that keeps the CAD kernel
  imported so the next build skips ~2.6 s of import. Started on demand, exits
  after an hour idle.
- **The CLI** (`cadgen <verb>`): `step build`, `step snapshot`,
  `stl|3mf|glb build`, `*snapshot`, `urdf|sdf|srdf validate`, `doctor`,
  `store`, `daemon`, `viewer`, `mcp`, `analytics`.
- **The CAD Viewer backend** (`cadgen/viewer`): a local HTTP server on port
  3245 that serves the viewer client and model data.
- **The MCP server** (`cadgen/mcp`): speaks MCP over stdio to agent apps and
  serves the CAD app page as cards (Claude Desktop) or tabs (Codex).

### 3. Shared JavaScript (`packages/core`, `packages/ui`)

- `@text-to-cad/core`: three.js-based CAD runtime with no React. It covers
  STEP/GLB/URDF loading, rendering, drawing and export. It also produces the
  Node builders and headless snapshot bundle that cadgen runs.
- `@text-to-cad/ui`: the React FileViewer, renderers (STEP, mesh, DXF, robot,
  GLB), tools and settings. Platform-agnostic. Hosts inject capabilities.

### 4. Apps (`apps/`)

| App | npm name | What it is | Ships where |
|-----|----------|------------|-------------|
| `apps/web` | `@text-to-cad/web` | CAD Viewer client (Vite + React) | Built into the wheel as `_runtime/viewer`, served by `cadgen viewer` |
| `apps/mcp` | `@text-to-cad/mcp` | CAD app page for MCP Apps hosts | Built into the wheel as `_runtime/mcp`, served by `cadgen mcp` |
| `apps/docs` | `cad-skills-docs` | Docs site + `/v1` analytics/version API (Next.js) | Vercel |

### 5. Plugin manifests (repo root)

The repository root *is* the plugin. `.claude-plugin/`, `.codex-plugin/`,
`.cursor-plugin/`, `gemini-extension.json` and the `*.mcp.json` files tell each
agent app where the skills are and how to start the `cad` MCP server.

### 6. Sample models (`models/`)

Hand-run fixtures and showcase projects (F1 car, turbofan, humanoids, gear
assemblies, drawings). **Automated tests never read them.** They exist for
manual review and demos.

## How a request flows

**"Make me an L-bracket with two M6 holes"**, in Claude Code with the plugin:

1. The agent matches the request to `skills/cad/SKILL.md` and reads it.
2. It writes `src/l_bracket.py` with an `@step(out="../STEP/l_bracket.step")`
   function using build123d.
3. It runs the script with the skill's pinned `python`. The decorator asks the
   warm daemon to build. The daemon runs the kernel and writes the STEP.
   Derived data goes into the store.
4. It optionally runs `cadgen step snapshot` to render a PNG and check its own
   work visually. That uses Playwright's headless Chromium and the `_runtime/browser`
   bundle.
5. The user says "show me". The agent calls the MCP tool `cad_show`. The host
   either renders a viewer card inline or, in a text-only host, returns a link to
   `http://127.0.0.1:3245/?file=...` served by `cadgen viewer`.

## Design principles worth knowing early

From `packages/cadgen/README.md` (19 laws) and `AGENTS.md`. These are enforced
by tests under `tests/python/global/`.

- Generated files are independent of their source. A STEP carries no
  pointer back to the script.
- The store holds only derived results and can always be deleted.
- Output is byte-deterministic.
- Loud failure or correct output. Nothing silently half-works.
- No backwards compatibility. Old formats get a migration message, not a shim.
- `packages/cadgen` ships alone. Its docs never reference repo paths.
- A skill never imports another skill or anything from the repo root.
- `VERSION` is the single source of truth. Release tooling stamps every other
  version string. Do not hand-edit them.
- No symlinks, no Git LFS, no file over 5 MiB in the tree. Agent installers
  clone the whole repo and some drop symlinks silently.

## Tech stack summary

| Layer | Technology |
|-------|-----------|
| CAD kernel | OpenCascade 7.9 via `cadquery-ocp-novtk`, `build123d` 0.11 |
| Python deps | ezdxf, matplotlib, pillow, shapely, playwright 1.63.0 |
| Native | One C file tracer, cross-compiled with zig 0.16 |
| JS runtime | three.js 0.186.1, three-mesh-bvh, meshoptimizer |
| UI | React 19, Radix UI, Tailwind 4, Vite |
| Docs | Next.js, shadcn, Postgres (Neon) for analytics, Vercel |
| Packaging | setuptools wheel, uv/uvx for end-user installs |
| CI | GitHub Actions (`test.yml`, `release-prepare.yml`, `release-publish.yml`, `deploy-docs.yml`) |
