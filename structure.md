# Minimalist Pipeline — Code Structure Map

Navigational map of every relevant source file: what each module owns, its key
classes/functions, and how the pieces connect. Complementary to `CODE.md`
(function inventory + patterns), `graph.md` (execution traces), and `NOTES.md`
(design rationale). This file answers "where do I look and what touches what?"

- Stack: Python 3.11+, Blender 4.2+ addon. Stdlib + `bpy` only (FFmpeg optional, farm video stages).
- Interaction philosophy: **Detect → Inform → Propose → Execute if validated**. No silent actions, no blockades.

---

## 1. Addon entry & registration

| File | Role |
|---|---|
| `__init__.py` | Addon root. `bl_info`, registration, WindowManager/Scene props, timers, teardown |
| `blender_manifest.toml` | Blender Extensions metadata (4.2+). Keep version in sync with `bl_info` manually |
| `addon_data.py` | `PipelineProjectItem` (opened-project list entry) + `PipelineAddonPreferences` (all settings + `draw()`) |

### Registration flow (`__init__.py:register`)

```
classes = *lib.classes + (PipelineProjectItem, PipelineAddonPreferences) + *panel_classes + *operator_classes + *menu_classes
```

- `_WM_PROPS` / `_SCENE_PROPS` — addon-props outside PropertyGroups, registered/unregistered by `_register_props()`/`_unregister_props()` (one loop, not per-site blocks).
  - WindowManager: `projects_collapse`, `entries_collapse`, `pipeline_entry_buffer` (CollectionProperty `PipelineEntryItem`), `shots_list_creation` (`PipelineShotItem`), `render_subshots`, `file_details_selected`, `file_selected_departments`, `entries_hide_done`, `entries_department_filter` (enum from `lib.department_filter_items`), `entries_min_version`.
  - Scene: `is_worker`, `pipeline_farm_list` (`PIPELINE_PG_farm_list_item`).
- Side-effect registrations (not classes): `register_topbar_menu()` (appends draw callback to `TOPBAR_MT_editor_menus`), `lib.register_handlers()` (4 handlers + heartbeat timer).
- Deferred one-tick timers (order matters, see `NOTES.md` "Addon register()"):
  - `lib.refresh_read_only_flag` (0.0)
  - `refresh_monitor_cache` (0.0)
  - `_deferred_project_check` (0.05) — deactivate unreachable active project (NAS/drive guard)
  - `_deferred_keymap` (0.1) — Ctrl+S override (`WM_OT_safe_save`)
  - `_deferred_auto_worker` (0.1) — auto-launch worker role if `auto_worker_on_open`
  - `_deferred_onboarding` (0.5) — first-launch welcome popup (skip in `bpy.app.background`)
- Startup: `lib.load_project_data()` restores opened projects + active project; `set_running_project(None)`; `register_status_timer()`.
- `unregister()` reverses: handlers, topbar menu, project backup save, classes (reversed), farm timers, monitor stop, `kill_worker()` if `is_blender_worker()`, props, shortcut override.

---

## 2. `lib/` — core logic (no UI classes except the shared popup operators)

| File | Owns |
|---|---|
| `errors.py` | `PipelineError(message, level)` — the one exception type; raised by core, caught once in operators |
| `actions.py` | `PipelineAction` dataclass (title/message/severity/choices/on_dismiss/explanation), `set_pending_action` / `get_pending_action` (global popup store) |
| `core.py` | Utilities: `addon_pref`, `resolve_ffmpeg`, `json_get`, `read_csv`, UI text helpers (`text_to_lines`, `lines_budget`, `list_to_labels`, `responsive_layout`, `draw_box_tip`, `region_char_budget`), `get_machine_id`, `random_display_name`, `path_reachable`, `resolve_bpy_path`, `now` **and the locking layer**: `acquire_lock` / `check_lock` / `release_lock` / `refresh_lock` (O_EXCL lock files, 90s staleness) + `locked_json()` context manager (atomic JSON read/write) |
| `config.py` | `ConfigCache` (cached, mtime-invalidated project config; `.get()`/`.invalidate()`/regular+shot regex/`.get_path()`), `parse_filename`, `sanitize_name`, shot segment helpers (`format_shot_segment`, `shots_in_segment`, `parse_timeline`, camera name build/parse), path routing (`prefix_to_parent_folder`, `type_by_folder`, `find_project_root`, `to_relative`/`to_absolute`, `file_in_active_project`, `find_known_project_for_file`) |
| `creation.py` | `create_asset_file`, `create_shot_file` (mono `int` or block `list[int]` shot numbers), `resolve_timeline`, constants `DEFAULT_ASSET_DEPARTMENTS` / `DEFAULT_SHOT_DEPARTMENTS` / `DEFAULT_MULTISHOT_STEP` |
| `browser.py` | Enum callbacks for dialogs' dropdowns: file cascade (`prefix_items`, `asset_folder_items`, `sequence_items`, `shot_items`, `version_items`, `dir_version_items`, `entry_version_items`, `shot_tag_items`) + department pickers (`asset_/shot_/file_/tracked_department_items`, `department_filter_items`) |
| `batch.py` | CSV batch: `parse_asset_batch_csv`, `parse_shot_batch_csv`, `resolve_batch_departments`, `launch_batch_create_entry` (per-row headless subprocess), `read_batch_result`, existence guards |
| `handlers.py` | Blender handlers: `post_load_handler` (session, library-update check, lock+heartbeat, read-only gate, auto-version), `on_quit_handler`, `save_post_handler`, `import_post_handler`, `heartbeat_30s` timer, `refresh_read_only_flag`, `register_handlers`/`unregister_handlers` |
| `session.py` | Identity (`get_user`, `get_user_data`), sessions (`session_update`, `close_session`, `scan_sessions`), project backup (`save_project_data`/`load_project_data`→`pipeline_backup.json`), `set_active_project_root` (single switching point; stops/launches farm roles), read-only flag (`get/set_opened_as_read_only`, `get_read_only_reason`) |
| `tracking.py` | The metadata layer — see section 3 |
| `libraries.py` | Append/link detection: `clean_append_and_relink`, `clean_append`, `import_warnings`, internal-root detection |
| `versioning.py` | `get_version_number`, `get_last_version_number`, `save_as(tag)` |
| `presets.py` | `install_default_preset`, `install_default_ffmpeg_preset`, `apply_asset_preset` (+ fallback), `build_shot_scene` (camera+marker scaffolding), `derive_shot_subranges` (markers→frame ranges read-back) |
| `preview.py` | `latest_shot_mp4`, `resolve_block_sources`, `resolve_sequence_sources` (filesystem-only mp4 providers for multishot/sequence previews) |
| `saving.py` | The Ctrl+S override: `WM_OT_safe_save`, `override_shortcut`/`unoverride_shortcut`, `get_save_shortcut` |
| `logs.py` | `log(level, origin, message)` → JSONL in `config/logs/`, rotation past `LOG_ROTATE_MAX_BYTES` |
| `operators.py` | Shared/cross-cutting operators: `M_PIPELINE_OT_action_popup` (generic popup running a pending `PipelineAction`'s callbacks; the single error safety net), `M_PIPELINE_OT_text_popup`, `M_PIPELINE_OT_onboarding_popup`, `M_PIPELINE_OT_auto_version` (open-time version proposal), `WM_OT_open_folder`, `M_PIPELINE_OT_current_frame` |
| `__init__.py` | Re-exports every public symbol; `classes` tuple = the 7 core operators |

### 3. `lib/tracking.py` — metadata layer in detail

Per-file JSON siblings of `.blend` files under `.pipeline/`:

```
assets/ch/ch_bob/.pipeline/
├── tracking.json          # departments_required, description, archived flag, entries
├── v001.wipmeta           # author, edited_by/at, departments worked, linked libs
└── v003-stable.stablemeta # departments_validated  (one per -stable version)
```

| Symbol | Purpose |
|---|---|
| `TrackingStatusCache` | Per-asset computed status, mtime-invalidated, `.get()` never raises |
| `WorkTimeCache` / `RecentFilesCache` | Read `sessions_log.jsonl`: total work time vs current-user recent files |
| `create_tracking` / `get_departments_required` | Init tracking.json / its department list |
| `create_wipmeta` / `wipmeta_touch` / `wipmeta_add_work` / `get_session_worked_departments` | WIP metadata lifecycle; save-time touch; worked-dept toggle state |
| `create_stablemeta` / `get_last_stable` / `set_department_validated` | Stable version metadata + in-place validation flips |
| `create_entry` / `edit_entry` / `delete_entry` / `get_entries` / `toggle_entry_task` | Review entries (with optional `shot` tag) |
| `group_entries_by_review` / `filter_review_boxes` / `_thread_meta` | Threaded review-box rendering helpers |
| `upload_csv` | Entry CSV import (`imported_count, total_rows`) |
| `check_library_update` / `library_updates` / `wipmeta_update_libraries` / `find_linked_by` | Linked-library staleness proposal + batch update |
| Branch/archive helpers | `copy_entries`, `archive_folder`, `is_block_archived`, `list_active_blocks`, `active_shot_owners` |
| `format_duration`, `department_status_tooltip`, `get/set_description`, `get_linked_libraries`, `clean_libraries` | Display + misc |

---

## 4. `operators/` — UI actions (one `try/except PipelineError` per `execute()`)

| File | Operators (M_PIPELINE_OT_*) |
|---|---|
| `project_ops.py` | `create_project`, `find_project`, `set_active_project`, `unset_active_project`, `remove_project`, `edit_project` |
| `asset_ops.py` | `create_asset` |
| `shot_ops.py` | `create_shot`, `edit_block_structure` (multishot "branch"), `add_multishot_item`, `remove_multishot_item` + `PipelineShotItem` PG + shared draw helpers (`_draw_shot_list`, `_draw_block_warning`, `_draw_timeline_warnings`, `_draw_naming_preview`, `_classify_shot_conflicts`) |
| `preview_ops.py` | `compile_preview` |
| `batch_ops.py` | `batch_create` (modal, polls headless subprocess) |
| `file_ops.py` | `increment_version` |
| `browser_ops.py` | `open_file`, `open_file_version` |
| `farm_ops.py` | `farm_request_render` (incl. block per-shot checklist), `farm_launch_monitor`, `farm_kill_monitor`, `farm_monitor` (dashboard popup), `farm_add_to_list`, `farm_list_delete`, `farm_add_self_worker`, `farm_kill_self_worker`, `farm_cancel_job`, `farm_archive_job` + PGs `PIPELINE_PG_farm_list_item`, `PIPELINE_PG_render_subshot_item` |
| `tracking_ops.py` | Entry CRUD (`create_entry`, `edit_entry`, `delete_entry`, `add/remove_entry_line`, `generic_entry_button`, `toggle_entry_task`), `upload_csv`, `tracking_monitor`, `edit_description`, `toggle_worked_department`, `toggle_validated_department`, `department_status_info`, `tracking_file_details` + `PipelineEntryItem` PG |

`operators/__init__.py` — the `classes` tuple only; a stale/renamed class missed here would fail silently (see `CODE.md` convention).

---

## 5. `panels/` — 3D viewport N-panel sidebar (Pipeline tab)

| File | Panel |
|---|---|
| `project_panel.py` | `M_PIPELINE_PT_project_panel` — own `draw_header()`; active project + create/find/edit/unset/remove |
| `file_panel.py` | `M_PIPELINE_PT_file_panel` — per-file actions (asset or shot): version, render, preview, branch (shot), open folder; worked-department toggle row; read-only awareness |
| `farm_panel.py` | `M_PIPELINE_PT_farm_panel` — own header with live monitor status, `DEFAULT_CLOSED`; `draw_farm_jobs`, `draw_farm_workers` |
| `tracking_panel.py` | no panel class — pure drawing helpers used by the tracking dialogs/`tracking_file_details`: `draw_tracking_data`, `draw_entries`, `draw_file_details`, `draw_monitor_table`, `_draw_entry`/`_draw_thread`, `TYPE_ICON`, filters |

No `bl_parent_id`, no shared wrapper panel — each collapsible independently.

## 6. `menus/` — top bar

| File | Content |
|---|---|
| `top_bar.py` | `M_PIPELINE_MT_topbar_menu` (condensed actions), `M_PIPELINE_MT_read_only_menu` ("READ-ONLY" reason + Increment), `read_only_indicator` (prepended label on `TOPBAR_MT_editor_menus`), `register`/`unregister` (menu + draw-callback side effects) |

`menu_register`'s draw callback + `override_shortcut` are the two side-effect registrations called explicitly from root `register()`.

---

## 7. `farm/` — distributed render (JSON-file coordination, no server)

Roles coordinated through `config/.farm/` on the shared drive (`monitor.lock`, `workers/`, `queue/{incomings,actives,archives,requests}/`).

| File | Owns |
|---|---|
| `loop.py` | `farm_tick` (both roles' shared timer), `register_farm_loop`, `stop_farm_role_for_project`, UI redraw timers: `_refresh_tick`/`register_refresh_timer` (full snapshot, only while dashboard open) + `_status_tick`/`register_status_timer` (status only, always on), snapshot builders (`_scan_active_jobs`, `_scan_incoming_jobs`, `_build_job_entry`, `_compute_snapshot`, `_read_monitor_status`) |
| `monitor.py` | Monitor role: `launch_monitor`/`is_blender_monitor`/`stop_monitor_loop`/`monitor_tick`, request writers `job_request` (one per submission), `request_preview_compile`, `request_monitor_kill`, `job_cancel_request`, `archive` (dashboard "✓"), `detect_orphaned_jobs` |
| `workers.py` | Worker role: `launch_worker`/`is_blender_worker`/`kill_worker`, heartbeats, `execute_render_request`, `apply_custom_preset`, process/cancel scanning |
| `queue.py` | Stage machine: `scan_requests` (incoming→active), `scan_queue` (advance one stage), `scan_processes` (Popen completion), `check_render_completion`, `mark_stage`, `construct_history`/`construct_split_history` |
| `dispatch.py` | `render_dispatch` (`single` vs `placeholder`/`auto` over shared drive), `render_mode_auto`, negative-collision tracking (`last_render_had_collision`, `append_render_history`), `local_slot_busy` |
| `setup.py` | Per-job output path (`compute_output_path`), context resolution (`resolve_job_context` — `shot_override` routing for block child jobs), headless setup launch (`run_render_setup`), in-subprocess `run_render_setup_entry` (_split_into_shot_jobs is where blocks split), `resolve_override_range`, `resolve_local_binary` |
| `post_render.py` | ffmpeg stages: `checks_images` (corruption pass), `compilation` (preset-driven video), `build_concat_command`/`run_preview_compile` (concat filter, never `-c copy`) |

### Job stage machine

```
queued → setup_start/finished/failed → render_start/finished/failed
      → checks_images_* → compilation_* → finished → archived
(+ orphaned on monitor startup; "archived" = history marker, file stays in actives/ until manual Archive)
Plain job:        queued → setup_finished → render_start → … → finished → archived
Multishot block:  queued → setup_finished → split_finished → archived
                  (fans out into one ordinary queued job per shot, shot_override set)
Preview compile:  queued → preview_queued → preview_start → preview_finished/failed → archived
```

---

## 8. `templates/` — bundled defaults & subprocess entry scripts

| File | Role |
|---|---|
| `asset_file_preset.py` | Copied to new projects as user-editable `config/presets/asset_file_preset.py`; `build_collections(prefix, name)` |
| `ffmpeg_preset_default.json` | Default ffmpeg preset, copied to `config/presets/ffmpeg_presets/default.json` |
| `batch_create_entry.py` | Headless per-row CSV batch creation (reads a request JSON, `read_homefile(use_empty=True)` first) |
| `farm_entry_render_setup.py` | Headless render-setup entry → `farm.run_render_setup_entry(job_id)` |
| `worker_render_entry.py` | Headless worker render → enables addon, applies custom preset, `bpy.ops.render.render(animation=True)` |

---

## 9. Persisted data on disk (relevant for customization)

| Path | Contents |
|---|---|
| `<project>/config/project_config.json` | Naming rules, structure/prefixes, departments, farm settings, resolution/fps/frames, `pipeline_addon_version` |
| `<project>/config/logs/pipeline_log.jsonl` / `sessions_log.jsonl` | Event log / session durations (work time) |
| `<project>/config/presets/` | User-editable collection preset + ffmpeg presets |
| `<project>/config/.sessions/.session_{pid}.json` | Live session heartbeat files |
| `<project>/config/.farm/` | Lock, worker registrations, job queue |
| `<file>.pipeline/tracking.json` + `{v}.wipmeta` + `{v}-stable.stablemeta` | Per-file metadata (see section 3) |
| Blender user config → `pipeline_backup.json` | `opened_projects` + `active_project_root` (via `session.py`) |
| `<file>.lock` | O_EXCL lock files beside any JSON being written (`locked_json`) |

---

## 10. Docs map (which file explains what)

| File | Content |
|---|---|
| `README.md` / `README_fr.md` | User-facing overview, concepts, installation |
| `CODE.md` | Dev function inventory + patterns + roadmap (v0.2: casting, build/rebuild; known tech debt) |
| `graph.md` | Execution traces of every handler/timer/operator |
| `NOTES.md` | Design rationale per feature (currently multishot deep-dive) |
| `README_EXTENSION.md` | Blender Extensions platform packaging |
| `docs/*.md` (+ `_fr` twins) | One page per theme: getting-started, project-and-naming, multishot, versions, sessions-and-locking, tracking-and-reviews, linking, farm, design, interface, limitations |
| `structure.md` | **This file** — navigation map of all relevant code |

---

## 11. Quick lookup: "I want to change X"

| Want to change | Go to |
|---|---|
| Addon settings / prefs UI | `addon_data.py` (`PipelineAddonPreferences`) |
| Sidebar panels layout | `panels/` (project/file/farm) + `panels/tracking_panel.py` draw helpers |
| Top bar menu / READ-ONLY indicator | `menus/top_bar.py` |
| A dialog/operation button | matching file in `operators/`, its enum items in `lib/browser.py` |
| File naming / folder routing | `lib/config.py` (regex, `prefix_to_parent_folder`) |
| JSON metadata schema | `lib/tracking.py` (+ `lib/versioning.py` for versions) |
| Locking / atomic JSON writes | `lib/core.py` (`locked_json` + lock primitives) |
| Open/save/quit behavior | `lib/handlers.py` + `lib/saving.py` (Ctrl+S) |
| Farm stages / jobs | `farm/queue.py` (state machine), `farm/monitor.py` (requests), `farm/setup.py` (blocks split here) |
| Multishot blocks | `operators/shot_ops.py` + `lib/presets.py` + `NOTES.md` |