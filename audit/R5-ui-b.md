# R5-ui-b audit notes (crates/ui-egui second half: panels + remaining UI + locales)

Auditor: subagent, 2026-10-13. Scope: audit/chunks/R5-ui-b.txt (68 files, ~26.2k lines) + skim of crates/ui-egui/locales/*.json.
Method: per-file grep for capability markers, then read. Tiers: 1 = touches A-L, 2 = ordinary logic, 3 = pure computation.

| path | lines | tier | caps | status | note |
|---|---|---|---|---|---|
| crates/ui-egui/src/panels/edit.rs | 1219 | 2 | none (dispatches commands via app.run; checks services.pick/save_curve_preset_file presence only) | done | Edit panel UI: sliders, histogram, curve, mixer, grading. Clean. |
| crates/ui-egui/src/panels/faces.rs | 780 | 1 | A(indirect: faces.models.download cmd after explicit accept; services.open_url on user click of "Open page"), G(idle detection: scan pace from focus/minimized/last input) | done | Download only via dialog with accept tickbox -> engine faces.models.download (engine-side, other chunk). Idle-aware pacing of LOCAL face indexing, only when user enabled recognition (faces.enable). No exfil; UI text says nothing uploaded. Info only. |
| crates/ui-egui/src/panels/filterbar.rs | 306 | 3 | none | done | Filter bar widgets -> library.filter commands. Clean. |
| crates/ui-egui/src/panels/grid.rs | 1132 | 2 | C(indirect, user-clicked: photo.delete / deletePermanently via context menu) | done | Grid layout, thumbnails, context menu dispatching commands. Instant used only for stats timing. Clean. |
| crates/ui-egui/src/panels/keymap.rs | 224 | 2 | D-check: reads egui key events only while user is recording a shortcut, in-app only; Paste payload discarded | done | Shortcut editor. Not a keylogger (egui window events, consumed only when recording). Clean. |
| crates/ui-egui/src/panels/keyword_list.rs | 621 | 3 | none (test reads locales/*.json via CARGO_MANIFEST_DIR) | done | Keyword tree UI. Recursion over keyword tree (catalog-bounded). Clean. |
| crates/ui-egui/src/panels/keywording.rs | 380 | 3 | none | done | Keyword chips. Pure UI/logic. Clean. |
| crates/ui-egui/src/panels/left.rs | 1521 | 1 | D(env HOME/USERPROFILE -> Pictures/Desktop/Downloads/Home listed in Local sidebar; read_dir of subfolders only when user expands a tree row, depth<=8, dir names only, hidden dot-dirs skipped), F-ish(worker threads per frame for missing-file count (Path::exists on library photos only) and fs listing), C(indirect, user-clicked folder.rename/folder.move/album.delete via dialogs) | done | Lightroom-style Local section. Nothing read beyond directory names / exists(); nothing sent anywhere. Album parent walks bounded (64). Clean. |
| crates/ui-egui/src/panels/library_problem.rs | 190 | 2 | none | done | Library-open error dialog. Clean. |
| crates/ui-egui/src/panels/masking.rs | 879 | 2 | A(indirect: offer_sam_download dialog when SAM3 missing; segmenter.remote_endpoint() checked - engine side) | done | Mask panel. SAM3 download only offered via dialog (known http-mirror issue in doc.md). Clean. |
| crates/ui-egui/src/panels/mod.rs | 101 | 3 | none | done | Module list, panel resize, toast. Clean. |
| crates/ui-egui/src/panels/notices.rs | 145 | 2 | C-check: persist/close_library on quit (own library) | done | Quit/unsaved prompts. Clean. |
| crates/ui-egui/src/panels/people.rs | 205 | 3 | none | done | People grid UI. Clean. |
| crates/ui-egui/src/panels/person.rs | 304 | 3 | none | done | Person page UI. Clean. |
| crates/ui-egui/src/panels/presets.rs | 235 | 2 | none (import/export via commands) | done | Preset list UI. Clean. |
| crates/ui-egui/src/panels/profiles.rs | 211 | 3 | none | done | Profile browser. Clean. |
| crates/ui-egui/src/panels/right.rs | 1167 | 1 | A/B(indirect: "Show on Map" button builds https://www.openstreetmap.org/?mlat=..&mlon=.. from photo GPS, opened via crate::links::open only on user click) | done | Right panel (crop/remove/info/keywords/versions/history). OSM URL is a fixed host, user-initiated, the user expects the browser to see coordinates. Info. |
| crates/ui-egui/src/panels/rules_editor.rs | 427 | 3 | G-check: "2026" literal is only a fallback year for a rule default (display), not a trigger | done | Smart-album rule editor; recursion depth capped at 3 for new groups. Clean. |
| crates/ui-egui/src/panels/second.rs | 56 | 3 | none | done | Second-window viewport. Clean. |
| crates/ui-egui/src/panels/strip.rs | 78 | 3 | none | done | Tool strip buttons. Clean. |
| crates/ui-egui/src/panels/settings.rs | 678 | 1 | B(indirect: stores user-typed external_editor app/path; executed elsewhere in menus.rs:1049, other chunk), C(user-chosen "Delete them" for smart previews when moving that cache), D(app.displayProfile reads an ICC file at a given path, read-only) | done | Settings dialog; all actions user-driven commands. Auto-Import watched folder only when user picks it. Clean. |
| crates/ui-egui/src/panels/topbar.rs | 192 | 2 | A(indirect: discord icon -> app.discord command -> links.rs DISCORD="https://discord.gg/artcraft", user click only) | done | Top bar, search field, window drag via egui viewport cmds. Clean. |
| crates/ui-egui/src/panels/unnamed.rs | 299 | 3 | none | done | Unnamed faces UI. Clean. |
| crates/ui-egui/src/pick.rs | 289 | 2 | (tests only: write/remove_dir_all under std::env::temp_dir lc-pick-<pid>) | done | Async file dialog plumbing; re-runs a command with picked path. Clean. |
| crates/ui-egui/src/region.rs | 546 | 3 | none | done | Pure loupe window geometry + tests. Clean. |
| crates/ui-egui/src/render.rs | 1353 | 2 | F-check: job pool worker threads (render only), idle trim calls engine gpu::trim_pool / memory::release (known sysmem malloc relief); Instant for idle timing | done | Render queue/texture management. Tests use temp_dir lc-ui-thumb-resize-<pid>. Clean. |
| crates/ui-egui/src/shortcuts.rs | 899 | 2 | D-check: reads egui key/Copy/Cut/Paste events of the app window to dispatch shortcuts; Paste payload not used | done | Keymap parse/dispatch; tests use fake /lightcraft-shortcuts paths in-memory. Clean. |
| crates/ui-egui/src/shortcuts/tests_relative_ratings.rs | 181 | test | none | done | In-memory synthetic key tests. Clean. |
| crates/ui-egui/src/softpaint.rs | 502 | 3 | D-check: rasterizes egui tessellated output of the APP ITSELF into an image (headless screenshots), not an OS screen grab; scoped threads only | done | Pure CPU triangle rasterizer, size-capped (16384 edge, 64 MP). Clean. |
| crates/ui-egui/src/state.rs | 945 | 3 | none (data types; settings.external_editor field stored in ui.json) | done | UI/app state structs + defaults + sanitize clamp. Note: UiState is settable via control channel per doc comment; external_editor exec path lives in menus.rs (other chunk) under the known opt-in control-server issue. Clean. |
| crates/ui-egui/src/sync.rs | 342 | 1 | F/C-check: worker threads for Synchronize Folder scan/run of a USER-CHOSEN library folder; "remove missing" moves catalog records to Recently Deleted (UI text: no file on disk touched); engine::sync does fs work (other chunk) | done | User-initiated, activity-tracked, cancellable. Clean. |
| crates/ui-egui/src/tasks.rs | 118 | 2 | F-check: generic worker-thread helper; "quiet" tasks (auto-import folder listing) run without an activity row - only for the user-configured watched folder | done | Clean. |
| crates/ui-egui/src/text_field.rs | 735 | 2 | D-check: clipboard only via egui Cut/Copy events and ViewportCommand::RequestPaste when the user picks Paste in the field menu | done | Text field widget; tests use HeadlessView fake clipboard. Clean. |
| crates/ui-egui/src/theme.rs | 343 | 3 | H-check: include_bytes! of assets/fonts/Inter-Regular.ttf & Inter-SemiBold.ttf - verified valid TrueType (17 standard tables, table data ends at file end, no appended payload) | done | Colour tokens, font setup, egui visuals. Clean. |
| crates/ui-egui/src/titlebar.rs | 76 | 3 | none (egui ViewportCommand StartDrag/Maximized only) | done | Clean. |
| crates/ui-egui/src/widgets.rs | 637 | 3 | none | done | Widget drawing + automation registry (in-memory widget id->rect for control channel). Clean. |
| crates/ui-egui/src/tests_activity.rs | 395 | test | C(test: writes/removes std::env::temp_dir()/lc-activity-*-<pid>) | done | Activity stack tests; synthetic DNG brackets; no network. |
| crates/ui-egui/src/tests_album_picker.rs | 128 | test | none | done | Headless UI test. |
| crates/ui-egui/src/tests_crop_rotate.rs | 366 | test | none | done | Headless UI test. |
| crates/ui-egui/src/tests_curve.rs | 154 | test | none | done | Headless UI test. |
| crates/ui-egui/src/tests_date_picker.rs | 125 | test | none | done | Headless UI test. |
| crates/ui-egui/src/tests_docking_acceptance.rs | 334 | test | none | done | Headless docking test. |
| crates/ui-egui/src/tests_filmstrip.rs | 30 | test | none | done | Headless UI test. |
| crates/ui-egui/src/tests_grid.rs | 157 | test | none | done | Synthetic in-memory library. |
| crates/ui-egui/src/tests_keymap.rs | 178 | test | none | done | Synthetic key events. |
| crates/ui-egui/src/tests_keyword_files.rs | 104 | test | C(test: temp_dir lc-ui-keyword-files-*-<pid>) | done | Keyword file import/export test. |
| crates/ui-egui/src/tests_keyword_list.rs | 430 | test | none | done | Headless UI test. |
| crates/ui-egui/src/tests_keyword_set.rs | 162 | test | none | done | Headless UI test. |
| crates/ui-egui/src/tests_keywording.rs | 229 | test | none (fake headless clipboard) | done | Headless UI test. |
| crates/ui-egui/src/tests_labels.rs | 85 | test | none | done | Render regression test. |
| crates/ui-egui/src/tests_library_problem.rs | 103 | test | C(test: temp_dir lc-ui-libproblem-<pid>); env LIGHTCRAFT_TEST_SHOTS opt-in screenshot dir | done | Test-only. |
| crates/ui-egui/src/tests_masking.rs | 1128 | test | C(test: temp_dir lc-ui-tree/-ext/-no-sam3-<pid>); asserts SAM3 does NOT download without consent | done | Test-only. |
| crates/ui-egui/src/tests_masking_layout.rs | 92 | test | none | done | Layout test. |
| crates/ui-egui/src/tests_menubar.rs | 147 | test | none | done | Headless UI test. |
| crates/ui-egui/src/tests_model_setup.rs | 202 | test | C(test: temp_dir lc-model-flow-*-<pid>); synthetic ONNX written locally; only downloadCancel invoked, no download | done | Test-only. |
| crates/ui-egui/src/tests_offline.rs | 184 | test | C(test: temp_dir lc-ui-slow-import-<pid>); sleeps simulate slow NAS | done | Test-only. |
| crates/ui-egui/src/tests_panels.rs | 1184 | test | C(test: temp_dir lc-ui-rename/lc-tracklog-toast-<pid>); GPX literal contains topografix namespace URI (XML ns, not fetched) | done | Test-only. |
| crates/ui-egui/src/tests_preview_limit.rs | 727 | test | none | done | Test-only. |
| crates/ui-egui/src/tests_quit_unsaved.rs | 128 | test | none (MemStore) | done | Test-only. |
| crates/ui-egui/src/tests_scroll.rs | 210 | test | none | done | Test-only. |
| crates/ui-egui/src/tests_switch_library.rs | 43 | test | C(test: temp_dir lc-ui-switchlib-<pid>) | done | Test-only. |
| crates/ui-egui/src/tests_sync.rs | 304 | test | C(test: Scratch temp_dir lc-ui-sync-*-<pid>, removed on Drop) | done | Test-only. |
| crates/ui-egui/src/tests_titlebar.rs | 117 | test | none | done | Test-only. |
| crates/ui-egui/src/tests_unsaved.rs | 81 | test | none | done | Test-only. |
| crates/ui-egui/src/tests_zoom.rs | 71 | test | none | done | Test-only. |
| crates/ui-egui/src/tests_zoom_keys.rs | 87 | test | none | done | Test-only. |
| crates/ui-egui/tests/docking_workflows.rs | 126 | test | env CRAFT_UI_DOCKING_DIR opt-in PNG output | done | Integration test (egui_kittest). |
| crates/ui-egui/tests/shutdown.rs | 40 | test | none | done | Integration test. |
| crates/ui-egui/locales/*.json (18 files) | ~1.18 MB | 3 | none | done (skim + scripted scan) | Flat {string: string} maps (230 keys per *-formats.json, 1600-2080 per language file). Scripted scan: no URLs/hosts, no script/shell text, no agent/LLM-directed text, no invisible/bidi/private-use/control characters; longest value 268 chars. Only "agent" regex hits were the colour name "Magenta". |

## Findings

No malicious or suspicious behaviour found in this chunk. All network, process, file-mutation and data-reading paths
are user-initiated UI actions that dispatch engine commands (implemented in other chunks). Informational items:

1. Info - faces.rs:22-48, 720-768 (`scan_pace`, `pump`): the background face indexer chooses its pace from window focus,
   minimisation and time since the last input ("full" when idle and the Faces settings are visible). Only runs after the
   user enables recognition (`faces.enable`) and installs a model; work is local face indexing of the user's catalog.
   Idle-awareness is for responsiveness, not concealment (progress is shown in Settings and the activity stack).
2. Info - right.rs:660-670: "Show on Map" builds `https://www.openstreetmap.org/?mlat={la}&mlon={lo}...` from the photo's
   GPS and opens it via `crate::links::open` only on a button click. Fixed host; expected behaviour.
3. Info - left.rs:351-360, 476-490: reads `HOME`/`USERPROFILE` to list Pictures/Desktop/Downloads/Home in the Local
   sidebar (Lightroom parity); `read_dir` of subfolder NAMES only when the user expands a row (dot-dirs skipped,
   depth <= 8). left.rs:298-340 stats `Path::exists` for library photos only (missing-file count). Nothing leaves the app.
4. Low/Info - settings.rs:140-146 + state.rs:148: `external_editor` is a free-text program/app name the user types; it
   is executed by "Edit in External Editor" in menus.rs:1049 (other chunk), which also accepts an `app` param. Anyone
   with access to the (opt-in, unauthenticated loopback) control channel could therefore pick the program to run - this
   is a consequence of the already-logged control-server issue, not new malicious code.
5. Info - topbar.rs:161: Discord icon runs `app.discord` -> links.rs `https://discord.gg/artcraft` on click only.

Checked and justified (not findings): theme.rs include_bytes! of Inter TTFs (verified valid TrueType, no appended
data); softpaint.rs renders the app's own egui output (no OS screen capture); keymap.rs/shortcuts.rs/text_field.rs read
egui window key/clipboard events only for shortcuts/paste the user triggers; sync.rs/tasks.rs/render.rs/left.rs worker
threads do render/scan/listing work for user-chosen folders; all test deletions are under std::env::temp_dir()/lc-*-<pid>;
no test downloads anything (tests_masking asserts SAM3 is NOT downloaded without consent).

No A-L capability found that is unjustified: no sockets, no Command/process spawn, no unsafe/FFI/asm, no link_section/
static initialisers, no date/time triggers, no obfuscated blobs, no persistence, no build scripts, no VM/debugger checks.

## Conclusion

Verdict for R5-ui-b: CLEAN. 68/68 assigned files fully reviewed (production files read line by line; test files
skimmed with targeted greps for network/delete/build-time behaviour) plus all 18 locale JSON files scanned.
