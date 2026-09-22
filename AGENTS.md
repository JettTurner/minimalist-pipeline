# AGENTS.md — For AI agents working in this repo

Agent onboarding doc. Read this first; then after it you need the source map
(`structure.md`), the dev function inventory (`CODE.md`), execution traces
(`graph.md`), and feature design rationale (`NOTES.md`).

## What this is

`Minimalist Pipeline` — a Blender addon (Python 3.11+, **Blender 4.2+** only)
for solo artists / small teams (2–5): naming, folders, versions, tracking,
and a distributed render farm that coordinates over a **shared drive via JSON
files** (no server, no SSH). Stdlib + `bpy` only; FFmpeg is the single optional
external dependency (farm video compile/check stages).

- Human overview + installation: `README.md`
- Re-verify version numbers stay in sync manually: `bl_info` in `__init__.py`
  **and** `blender_manifest.toml`.

## Orientation — read in this order

1. `structure.md` — map of every relevant file, what each module owns, the
   "I want to change X → go to" lookup table.
2. `CODE.md` — function inventory + code patterns + roadmap/known tech debt.
3. `graph.md` — exact trigger→decision-tree traces of every handler/timer/operator.
4. `NOTES.md` — the *why*: design history, rejected alternatives (currently a
   multishot deep-dive; grows feature by feature).
5. `docs/` — one user-facing page per concept (`README.md` table indexes them).

## Architecture in one breath

```
__init__.py      registration + props + timers (+ defer-once startup checks)
addon_data.py    preferences + project list
lib/             all core logic (config, creation, tracking, versioning, locks…)
operators/       UI actions (M_PIPELINE_OT_*)
panels/          N-panel sidebar (Project / File / Farm) + tracking draw helpers
menus/           top bar "Pipeline" menu + READ-ONLY indicator
farm/            monitor + worker roles, job stage machine, dispatch, ffmpeg
templates/       bundled presets + headless subprocess entry scripts
```

Registration is centralized: each sub-package exposes a `classes` tuple;
`__init__.py` assembles them with splats (`*lib.classes`, `*operator_classes`…).
Never hand-duplicate a class list. New WM/Scene props go in the declarative
`_WM_PROPS` / `_SCENE_PROPS` dicts in root `__init__.py`.

## Non-negotiable conventions

- **Interaction pattern**: Detect → Inform → Propose → Execute *if validated*.
  Never act silently, never block work. Automations propose via
  `PipelineAction` + `M_PIPELINE_OT_action_popup` (`lib/actions.py`,
  `lib/operators.py`).
- **Error contract**: core functions `raise PipelineError`, operators catch it
  **once** with one `try/except PipelineError` in `execute()` (log + `self.report`,
  return `{"CANCELLED"}`), and report success too (`{"INFO"}`).
- **Must NEVER raise** (Blender gives no error channel; an exception silently
  kills a timer forever): handlers, `bpy.app.timers` callbacks, `draw()`,
  enum-items callbacks. Catch internally, `log()`, return a safe default
  (see `session_update`, `check_library_update`, `TrackingStatusCache.get`).
- **`locked_json` commits on normal exit only.** A `raise` inside its `with`
  skips the pending write. Failure writes go through their own already-closed
  `locked_json` call — reuse `farm/queue.py`'s `mark_stage()`, never write-then-raise
  inline.
- `pathlib.Path` everywhere, never `os.path`. `pathlib` also doesn't resolve
  `//` — use `resolve_bpy_path()` for Blender-relative filepaths.
- Type hints: annotate context params as `bpy.types.Context`, never
  `bpy.context` (it's a live/restricted instance at import time and
  `bpy.context | None` raises `TypeError` on addon load).
- Docstrings: one concise line, English. Note when a function must never raise.
- UI: status bar for automatic actions, modal popup for user decisions.

## Working on the farm (where the sharp edges are)

- Roles coordinate purely through `config/.farm/` (`monitor.lock`, `workers/`,
  `queue/{incomings,actives,archives,requests}/`) — never direct connections.
- Job stage machine lives in `farm/queue.py` (`scan_requests`/`scan_queue`/
  `scan_processes`); all job kinds (plain render, multishot split, preview
  compile) share one flat `elif` chain keyed on stage name.
- A multishot block splits **at setup time**, in `farm/setup.py`
  (`run_render_setup_entry` → `_split_into_shot_jobs`), never at submission.
- `"archived"` is a stage-history marker, not a file move; files stay in
  `queue/actives/` until the dashboard's Archive button (`farm/monitor.py:archive`).
- All farm UI drawing is kept cheap: full snapshot timer only runs while the
  dashboard popup is open; a status-only timer always runs
  (`farm/loop.py`).

## Multishot (the feature an agent will most likely mis-touch)

- A mono-shot is a block of length 1 — **no special-case code path anywhere**
  (naming, creation, render split, camera scaffolding). Don't add one.
- Shot segment regex: `\d+(-\d+)*`, always sorted/deduped/zero-padded.
- Camera/marker scaffolding: `build_shot_scene` (write) / `derive_shot_subranges`
  (read-back, only ever once the file is genuinely open — never from filenames).
- `skipped_shots` / `absorbed_shots` are surfaced never silently, on the parent
  job and in the block's `.pipeline/render_history.json`.
- Preview compiles are **manual only**, never auto-triggered by render completion.
- Full rationale + spec pointers: `NOTES.md`.

## Verification / testing

- **No test suite, no lint config, no CI** in this repo. The addon doesn't run
  outside Blender (`bpy` import fails without it), so most of the code can't be
  executed here.
- What you *can* do: syntax-check pure-Python modules:
  `python -m py_compile lib/config.py lib/core.py farm/setup.py`
- Much of the environment lacks Blender — a note in `CODE.md` says multishot
  was built against standalone-Python checks only; flag code as **unverified
  against real bpy** when that's the case.
- If a change is needed and you can't run Blender, prefer edits that are
  locally checkable (regex/path/frame arithmetic stayed standalone for a reason).

## Project-specific cautions

- Don't put `os.path` next to `pathlib`; keep paths through
  `ConfigCache.get_path()` / `to_relative()` / `to_absolute()`.
- Read-only state: consult `get_opened_as_read_only()`/`get_read_only_reason()`
  (`lib/session.py`); the read-only flag is only ever *overwritten*, never
  cleared on open — always pair with `== bpy.data.filepath`.
- Active project switching must go through `lib.set_active_project_root()`
  (it handles farm-role start/stop), never assign `prefs.active_project_root`
  directly.
- Agents don't commit/push unless told to. If asked, inspect `git status`,
  `git diff`, and recent `git log` first.

## Roadmap / known gaps (see `CODE.md` for detail)

- v0.2: casting JSON (`sq\d+_sh\d+_casting.json`), build/rebuild (link assets
  into shots, dry-run first), MAX_PATH mitigation for block `.wipmeta` files.
- Known tech debt: `create_clean` uses `read_homefile()` (resets the live
  session); "monitor" names both the farm role and the UI popups on purpose;
  log rotation archives wholesale (fine at realistic scale).
- Don't build these without checking `CODE.md` first — the debt notes explain
  why they're deferred.

---

## If you're customizing

- The codebase was authored around strict conventions (above) — keep style
  consistent even when the features themselves change.
- Changes that reach `bpy` can't be executed here; prefer edits that are
  standalone-Python checkable and say so when a change is unverified.