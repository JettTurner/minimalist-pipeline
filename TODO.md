# TODO — Missing usability features

Candidate backlog derived from a review of the current addon (see
`docs/limitations.md`, `CODE.md` roadmap, and a code sweep). Items are
categorized by impact; each entry links the code it would touch. Revisit
priorities before starting work — some overlap, and a few only make sense
together (e.g. casting + build/rebuild are one feature).

## Highest impact (genuinely missing)

- [ ] **Delete / rename / retract an asset or shot**
  - The addon can create files, folders, and tracking on the shared drive,
    but there is no UI to undo a mistake. Only the project can be removed
    (`operators/project_ops.py` — and it explicitly keeps files on disk).
  - Touch points: new operators (delete/rename) in `operators/asset_ops.py` /
    `operators/shot_ops.py`, `CreationCache`/path helpers in `lib/config.py`,
    panels in `panels/file_panel.py`.
- [ ] **Force-unlock for stale/locked files**
  - `lib/core.py`'s `acquire_lock` auto-steals locks older than
    `LOCK_STALE_SECONDS` (90s), but the artist can't click "force unlock" —
    they must wait it out. A small operator calling `release_lock`
    (+ popup with clear warnings) would remove that wait.
- [ ] **Guard ALL save paths, not just Ctrl+S**
  - `WM_OT_safe_save` (`lib/saving.py`) only intercepts Ctrl+S. The top-bar
    save icon and File menu Save call `bpy.ops.wm.save_mainfile()` directly
    and bypass the read-only/stable checks (`docs/limitations.md`).
  - Touch points: `menus/top_bar.py` (save icon), `lib/saving.py`.

## Farm usability

- [ ] **Render-completion notifications**
  - The status timer always runs but never *tells* the user a job finished;
    they must open the dashboard. A notification popup / status-bar message
    keyed off job state changes in `farm/loop.py` / `farm/queue.py`.
- [ ] **Bulk queue operations** — cancel-all / retry-failed / archive-failed
  - Today cancel and archive are per-job (cancel is even per-job-per-worker).
    Touch points: `operators/farm_ops.py` (`farm_cancel_job`,
    `farm_archive_job`), `farm/monitor.py`.
- [ ] **Continuously re-check the worker**
  - `auto_worker_on_open` only fires at startup and on project switch
    (`docs/limitations.md`); no re-launch if the worker dies mid-session.
    Touch points: `lib/handlers.py` heartbeat, `farm/workers.py`.

## Versioning / review

- [ ] **Version comparison** (thumbnails, frame diff, or side-by-side open)
  - Versions are managed purely by filename today. Touch points:
    `lib/versioning.py`, `panels/file_panel.py`, new UI (thumbnails/cache).
- [ ] **Un-publish a `-stable` tag**
  - No way to revert a stable tag except opening and saving over it (with the
    existing warning). Touch points: `lib/versioning.py`, `lib/tracking.py`.
- [ ] **Project-wide search**
  - Sidebar only has a department filter + "hide done". A search box over
    assets/shots/entries. Touch points: `panels/tracking_panel.py`,
    `lib/tracking.py` (`TrackingStatusCache`), browser helpers in
    `lib/browser.py`.

## Roadmap ties (`CODE.md` v0.2)

- [ ] **Casting + build/rebuild** (casting JSON + linking assets into shots,
      dry-run first) — the single biggest content-level gap. Build with
      `CODE.md`'s existing notes and `NOTES.md` rationale in mind.

---

## Working notes

- **No test suite, no lint, no CI** — edits that reach `bpy` can't be executed
  locally; prefer standalone-Python-checkable changes and flag anything
  unverified against real Blender.
- Keep the interaction pattern: **Detect → Inform → Propose → Execute if
  validated**. Never silent, never blocking.
- Before starting any item, re-read `AGENTS.md` conventions and the relevant
  docs (`CODE.md`, `graph.md`, `NOTES.md`).