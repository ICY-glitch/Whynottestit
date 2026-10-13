# Findings (full audit, workflow `wf_573e1d86-7f2`)

All findings reported by the 11 review/test agents. Every finding first rated Medium or worse was checked by an independent verifier agent told to refute or downgrade it; the *Final* column is after verification. Evidence quotes are in each chunk's notes file (`audit/<chunk>.md`).

| # | Final | Reported | Verifier | Chunk | Where | Finding |
|---|---|---|---|---|---|---|
| 1 | **Low** | Medium | downgraded | R1-entry-network-tooling | `apps/lightcraft/src/main.rs:444` | Control channel / MCP --connect can launch an arbitrary executable via photo.editInExternal `app` parameter |
| 2 | **Low** | Low | n/a (Low/Info) | R1-entry-network-tooling | `crates/fetch/src/lib.rs:359` | Downloader follows https->http redirects; unpinned SAM 3 tokenizer files could be tampered with in transit |
| 3 | **Low** | Low | n/a (Low/Info) | R1-entry-network-tooling | `packaging/windows/sign.ps1:249` | Signing step downloads unpinned Microsoft.Trusted.Signing.Client and loads it with signing secrets in env |
| 4 | **Low** | Medium | downgraded | R2-engine-a | `crates/engine/src/cmd/keywords.rs:448` | Export-to-path commands overwrite any file without the originals guard; keyword.export lets a caller write chosen text lines to any file |
| 5 | **Low** | Low | n/a (Low/Info) | R2-engine-a | `crates/engine/src/cmd/browse.rs:223` | folder.rename / folder.move act on any absolute folder, not only folders the library knows |
| 6 | **Low** | Low | n/a (Low/Info) | R2-engine-a | `crates/engine/src/cmd/lut_profiles.rs:112` | LUT import without a library writes a predictable file in shared temp; delete check is lexical |
| 7 | **Low** | Low | n/a (Low/Info) | R3-engine-b | `crates/engine/src/preset_import.rs:64` | Zip inflate limit trusts attacker-declared uncompressed size (memory DoS on preset import) |
| 8 | **Low** | Medium | downgraded | R4-ui-a | `crates/ui-egui/src/menus.rs:1049` | Edit in External Editor runs any executable named in the `app` param (UI side of R1 finding F1) |
| 9 | **Low** | Low | n/a (Low/Info) | R4-ui-a | `crates/ui-egui/src/control.rs:293` | A control-channel client can read the system clipboard via ui.clipboard paste |
| 10 | **Low** | Low | n/a (Low/Info) | R5-ui-b | `crates/ui-egui/src/panels/settings.rs:144` | External editor setting is a free-text program executed elsewhere (reachable via the known control channel issue) |
| 11 | **Low** | Low | n/a (Low/Info) | R6-catalog-meta-ai | `crates/catalog/src/keywords.rs:284` | Unbounded recursion on hierarchical keyword depth from imported XMP (stack overflow) |
| 12 | **Low** | Low | n/a (Low/Info) | S14-dependencies | `crates/meta/src/xmp_merge.rs:117` | quick-xml 0.38.4 (RUSTSEC-2026-0194, quadratic duplicate-attribute check) is reachable when merging XMP sidecars |
| 13 | **Low** | Low | n/a (Low/Info) | S14-dependencies | `crates/fetch/Cargo.toml:15` | Model-download TLS uses an alpha, unaudited crypto provider (rustls-rustcrypto 0.0.2-alpha), which pulls in advisory-flagged crates |
| 14 | **Info** | Info | n/a (Low/Info) | R1-entry-network-tooling | `apps/lightcraft-web/src/web.rs:230` | Web host open_url lacks the https-only check the desktop host has |
| 15 | **Info** | Info | n/a (Low/Info) | R1-entry-network-tooling | `packaging/windows/lightcraft.wxs:10` | Packaging metadata copied from sibling apps does not match LightCraft |
| 16 | **Info** | Info | n/a (Low/Info) | R1-entry-network-tooling | `xtask/src/main.rs:117` | xtask probes system RAM / load average (dev tooling only) |
| 17 | **Info** | Info | n/a (Low/Info) | R2-engine-a | `crates/engine/src/cmd/convert.rs:292` | photo.duplicate TOCTOU between exists() check and fs::copy |
| 18 | **Info** | Info | n/a (Low/Info) | R2-engine-a | `crates/engine/src/cmd/query.rs:111` | Command clients can read local file contents back through import and metadata commands |
| 19 | **Info** | Info | n/a (Low/Info) | R3-engine-b | `crates/engine/src/memory.rs:46` | Runs sysctl / reads /proc/meminfo to learn RAM size |
| 20 | **Info** | Info | n/a (Low/Info) | R3-engine-b | `crates/engine/src/library.rs:344` | Library prefs.json can move the smart-previews folder elsewhere |
| 21 | **Info** | Info | n/a (Low/Info) | R4-ui-a | `crates/ui-egui/src/panels/denoise.rs:45` | In-app input timestamps pace background AI denoise (benign) |
| 22 | **Info** | Info | n/a (Low/Info) | R5-ui-b | `crates/ui-egui/src/panels/faces.rs:22` | Background face indexing adjusts its pace from idle time and window focus |
| 23 | **Info** | Info | n/a (Low/Info) | R5-ui-b | `crates/ui-egui/src/panels/right.rs:666` | Show on Map opens OpenStreetMap with the photo's GPS coordinates when clicked |
| 24 | **Info** | Info | n/a (Low/Info) | R5-ui-b | `crates/ui-egui/src/panels/left.rs:351` | Local sidebar reads HOME/USERPROFILE and lists folder names under it |
| 25 | **Info** | Info | n/a (Low/Info) | R6-catalog-meta-ai | `crates/catalog/src/lock.rs:73` | Host name, pid and program written to the library folder's lock-owner file |
| 26 | **Info** | Info | n/a (Low/Info) | R6-catalog-meta-ai | `crates/catalog/examples/persist_bench.rs:78` | Dev benchmark example wipes the directory passed on its command line |
| 27 | **Info** | Info | n/a (Low/Info) | R6-catalog-meta-ai | `crates/meta/src/lib.rs:12` | meta crate root lacks the panic-deny attribute; a few infallible expect() calls and direct indexing |
| 28 | **Info** | Info | n/a (Low/Info) | R7-raw-tiff | `crates/raw/src/vendor/sr2.rs:52` | Embedded XOR keystream seed for Sony SR2SubIFD decryption (justified) |
| 29 | **Info** | Info | n/a (Low/Info) | R7-raw-tiff | `crates/raw/src/vendor/arw.rs:89` | Sony maker-note byte-substitution decipher table (justified) |
| 30 | **Info** | Info | n/a (Low/Info) | R7-raw-tiff | `crates/raw/src/vendor/cr3.rs:446` | Literal NUL/control bytes inside test byte-string literals make sources look binary |
| 31 | **Info** | Info | n/a (Low/Info) | R8-compute | `crates/raster/examples/blur_bench.rs:14` | Only unsafe in chunk: libc clock_gettime in a dev-only benchmark example |
| 32 | **Info** | Info | n/a (Low/Info) | R8-compute | `crates/gpu/src/backend.rs:163` | GPU-init crash sentinel writes/removes a marker file at a caller-supplied path |
| 33 | **Info** | Info | n/a (Low/Info) | R8-compute | `crates/gpu/src/ctx.rs:293` | GPU adapter selection skips software (CPU) adapters; adapter info is used locally only |
| 34 | **Info** | Info | n/a (Low/Info) | S1-sweeps-binaries | `crates/catalog/src/lock.rs:73` | Library lock file records the computer's hostname |
| 35 | **Info** | Info | n/a (Low/Info) | S1-sweeps-binaries | `apps/lightcraft/src/main.rs:437` | Windows release binary still contains the known cmd /C start file-open path |
| 36 | **Info** | Info | n/a (Low/Info) | S14-dependencies | `Cargo.lock` | Unmaintained dependencies: paste 1.0.15 and ttf-parser 0.25.1 |
| 37 | **Info** | Info | n/a (Low/Info) | S14-dependencies | `Cargo.lock` | Dependencies ship prebuilt binary objects (standard upstream practice) |
| 38 | **Info** | Info | n/a (Low/Info) | S14-dependencies | `Cargo.lock` | Environment / container detection in a dev-only dependency (is-docker, is-wsl via open) |
| 39 | **Info** | Info | n/a (Low/Info) | S15-dynamic-run | `audit/S15-dynamic-run.md` | Dynamic run: no outbound network, no child processes, no bait access, no VM checks (behaviour matched expectations) |
| 40 | **Info** | Info | n/a (Low/Info) | S15-dynamic-run | `crates/engine/src/devices.rs:31` | Camera-card detection lists /media and /mnt and stats <mount>/DCIM in a background thread |
| 41 | **Info** | Info | n/a (Low/Info) | S15-dynamic-run | `audit/S15-dynamic-run.md` | Library lock owner file records the machine's host name |
| 42 | **Info** | Info | n/a (Low/Info) | S15-dynamic-run | `apps/lightcraft/src/main.rs` | Startup-failure path runs zenity for an error dialog; libX11 falls back to TCP 127.0.0.1:6000 |

## Details

### 1. [Low] Control channel / MCP --connect can launch an arbitrary executable via photo.editInExternal `app` parameter

- **Chunk:** R1-entry-network-tooling · **Where:** `apps/lightcraft/src/main.rs:444` · **Category:** B - Running other programs
- **Evidence:** main.rs:426-446: `open_with: Some(Box::new(|path: &str, app: &str| { ... } else { std::process::Command::new(app) }; c.arg(path).spawn()` ; crates/ui-egui/src/menus.rs:1049: `let editor = p.get("app").and_then(Value::as_str).map(str::to_string).unwrap_or_else(|| app.ui.settings.external_editor.clone());` ; crates/ui-egui/src/control.rs:177: `"engine.execute" | "ui.menu.invoke" | "command" => { ... app.run(id, params)`
- **Explanation:** Any local process that can reach the opt-in, unauthenticated 127.0.0.1 control port (or an MCP agent using `lightcraft-cli mcp --connect`) can send {command:"photo.editInExternal", params:{app:"/any/binary"}} and the app spawns that program, passing the edit-copy path as its argument. On macOS it becomes `open -a <app>`. The known issue only covered access to app commands; this lets the control server run any program on the machine. It looks like a design gap, not malicious intent. Possible fixes: have the control channel ignore `app` and only use the editor set in Settings, or require the user to confirm.
- **Verifier (downgraded, final Low):** The code path is real. control.rs:177-181 routes "engine.execute"/"ui.menu.invoke"/"command" to app.run(id, params). lib.rs:414-418 sends that to menus::run_ui_command. In menus.rs:1035-1056, "photo.editInExternal" first renders an edit copy, then takes `p.get("app")` from the request (falling back to the external editor set in Settings) and calls services.open_with. In apps/lightcraft/src/main.rs:426-446, open_with runs `Command::new(app).arg(path).spawn()` on Linux and Windows, or `open -a app path` on macOS. So whoever controls the request picks the program to run.

Why I lowered it from Medium to Low:
(1) The control server is off by default. It only starts with --control or LIGHTCRAFT_CONTROL_PORT (main.rs:708-716, 823) and only listens on 127.0.0.1 (control_server.rs:28).
(2) A browser cannot use it as a cross-protocol attack path. The server closes the connection at the first line that is not a JSON object with a `method` field, which includes an HTTP request line (control_server.rs:3-7, parse_line at 79-101).
(3) The channel is meant to be a full-trust automation interface. It already lets a client type and click in the UI (ui.key, ui.pointer, ui.clickWidget), so a client could set the external editor field in Settings (panels/settings.rs:144) and then trigger the same spawn. The `app` parameter is a shortcut to something the channel can already do, not a new capability.
(4) The attacker does not control the arguments: the only argument is the path of the rendered edit copy. A process running as the same user gains nothing it could not do itself. The remaining risk is another local user on a shared machine reaching the loopback port while the opt-in server is running, or an MCP agent going beyond what it was meant to do.

There is no sign of malicious intent; this is a hardening gap. A fix would be to ignore `app` when the request comes from the control channel, or to ask the user to confirm.

### 2. [Low] Downloader follows https->http redirects; unpinned SAM 3 tokenizer files could be tampered with in transit

- **Chunk:** R1-entry-network-tooling · **Where:** `crates/fetch/src/lib.rs:359` · **Category:** A - Network
- **Evidence:** lib.rs:353-360 follows 301/302/303/307/308 via `url = url.join(loc)`; http.rs:112-113 `if l.starts_with("http://") || l.starts_with("https://") { return Url::parse(l); }` ; crates/segment/src/fetch/mod.rs:20-21 `FileSpec { name: "vocab.json", size: None, sha256: None, max: 16 << 20 }`, same for merges.txt
- **Explanation:** A redirect can move the download from HTTPS to plain HTTP. The 3.4 GB weights are protected by a SHA-256 pin, and the bearer token is never sent to a redirected host or over http. The two tokenizer files have no pin, so an on-path attacker could swap them; they are capped at 16 MB and only parsed. This is related to the known 'http mirrors allowed for SAM3' item. Refusing https->http downgrades would close it.

### 3. [Low] Signing step downloads unpinned Microsoft.Trusted.Signing.Client and loads it with signing secrets in env

- **Chunk:** R1-entry-network-tooling · **Where:** `packaging/windows/sign.ps1:249` · **Category:** J - Build-time execution
- **Evidence:** Invoke-WebRequest -Uri 'https://www.nuget.org/api/v2/package/Microsoft.Trusted.Signing.Client' -OutFile $pkg ... Invoke-SignTool ($common + @('/tr', $ts, '/dlib', $dlib.FullName, '/dmdf', $metadata) + $Files)  (AZURE_CLIENT_SECRET etc. in env)
- **Explanation:** This is the same class as the known 'unpinned CI tool downloads' issue, but here the unversioned, unhashed DLL runs with access to the Azure code-signing credentials. Separately, packaging/linux/package.sh:113 downloads appimagetool from the `continuous` tag without a hash and embeds its runtime in the AppImages we ship. These are supply-chain hardening points, not malicious code.

### 4. [Low] Export-to-path commands overwrite any file without the originals guard; keyword.export lets a caller write chosen text lines to any file

- **Chunk:** R2-engine-a · **Where:** `crates/engine/src/cmd/keywords.rs:448` · **Category:** C File destruction/tampering
- **Evidence:** keywords.rs:448 `std::fs::write(path, &file.text).map_err(...)` (keyword.export, path from params, enabled `always`); preset_files.rs:75 `std::fs::write(&path, to_lcpreset(&chosen))` (preset.export); curves.rs:355 `std::fs::write(&path, to_file(&chosen))` (curve.exportPresets). In contrast cmd/export.rs:87 and cmd/merge.rs:33 call `s.check_write_target(path)` first. The keyword list writer (catalog/src/keywords.rs:464) emits `format!("{tabs}{}\n", n.name)`, and keyword names exclude only control chars, leading/trailing spaces, [..] and {..}.
- **Explanation:** Any MCP or control-channel client can overwrite any user-writable file, including a library original (no Unique/Skip policy, no OriginalGuard). For keyword.export the caller also controls the content: keyword.create {name:"<shell line>"} then keyword.export {path:"~/.bashrc"} writes those lines into a shell startup file, which can lead to code execution. The channel is opt-in (the loopback control server is already logged as unauthenticated; browser cross-protocol lines are reportedly rejected) or an MCP agent. This is not malicious, but it is a real hardening gap. Fix: route these writes through check_write_target, and refuse to overwrite an existing file or force the expected extension.
- **Verifier (downgraded, final Low):** The code is as described, but the severity is overstated and none of it is malicious. crates/engine/src/cmd/keywords.rs:448 does call std::fs::write(path, &file.text) on a caller-supplied path with no check_write_target. So do preset_files.rs:75 (preset.export) and curves.rs:355 (curve.exportPresets), and those two also create_dir_all the parent directory. In contrast, merge.rs:33, export.rs:87/89/122, control.rs:385/394 (ui.screenshot/ui.render) and mcp headless.rs:142 all call Session::check_write_target (originals.rs:89). So the originals guard really is applied unevenly.

The content control is real but limited. The writer (catalog keywords.rs:464) emits `{tabs}{name}\n`. Names can't contain '|' (SEP, keywords.rs:20), control characters, or bracketed/braced forms. They can still contain ';', '&&' or '$(...)', so a top-level keyword could become a shell line in a file like ~/.bashrc.

Why I downgraded:
1. These are deliberate "save to the path the user chose" features. The desktop UI calls them with a path from a file dialog (menus.rs:1467, 1517, 1578), like any Save As.
2. Reaching them without the UI requires the opt-in control channel or an MCP agent. Either one already drives the whole command surface on the user's behalf, and an MCP agent normally has its own filesystem access. The write gives no privilege the caller doesn't already have, and the account and permissions are the user's own.
3. preset.export and curve.exportPresets add an extension only when the path has none. Their content is structured preset data, so they're weaker write primitives than keyword.export.

The concrete defect is a hardening gap. These three writers skip OriginalGuard, so a careless or hostile caller could overwrite a catalogued original or sidecar. Routing them through check_write_target would fix that. It's a defense-in-depth or data-loss issue, not a Medium-severity file-tampering finding.

### 5. [Low] folder.rename / folder.move act on any absolute folder, not only folders the library knows

- **Chunk:** R2-engine-a · **Where:** `crates/engine/src/cmd/browse.rs:223` · **Category:** C File destruction/tampering
- **Evidence:** browse.rs:171-196 the folder.rename / folder.move commands take `path` from params, enabled `always`; rename_folder_on_disk (223-250) calls `std::fs::rename(src, dst)` after checking only `!dst.exists()`, `!dst.starts_with(src)` and (in move_folder:285) that it is not the library's own dir.
- **Explanation:** A remote command client can move or rename arbitrary user folders, e.g. ~/Documents. Nothing is overwritten or deleted and the step can be undone, so this is a Lightroom-parity feature with no scope restriction rather than anything malicious.

### 6. [Low] LUT import without a library writes a predictable file in shared temp; delete check is lexical

- **Chunk:** R2-engine-a · **Where:** `crates/engine/src/cmd/lut_profiles.rs:112` · **Category:** C File destruction/tampering
- **Evidence:** lut_profiles.rs:112-113 `let f = std::env::temp_dir().join(format!("lightcraft-{}.cube", ...)); std::fs::write(&f, &text)`; lut_profiles.rs:142 `if profiles_dir(s).is_some_and(|d| Path::new(&gone.file).starts_with(d)) { let _ = std::fs::remove_file(&gone.file); }`
- **Explanation:** On a shared /tmp, a predictable file name written with fs::write (which follows symlinks) could be redirected by another local user. The delete check is lexical, so a tampered prefs entry like Profiles/../x would pass it. The slug is restricted to [a-z0-9-], so normal use is safe. This is hygiene only.

### 7. [Low] Zip inflate limit trusts attacker-declared uncompressed size (memory DoS on preset import)

- **Chunk:** R3-engine-b · **Where:** `crates/engine/src/preset_import.rs:64` · **Category:** robustness
- **Evidence:** 8 => miniz_oxide::inflate::decompress_to_vec_with_limit(data, usize_.max(1) * 2 + 1024)  // usize_ = u32 read from the zip central directory
- **Explanation:** A crafted preset .zip or .mplumpack (the Luminar pack format) can declare a huge uncompressed size, so the importer may allocate up to about 8 GiB per entry when the user imports it. That can exhaust memory. It cannot run code or touch files, so it is a robustness issue, not malice.

### 8. [Low] Edit in External Editor runs any executable named in the `app` param (UI side of R1 finding F1)

- **Chunk:** R4-ui-a · **Where:** `crates/ui-egui/src/menus.rs:1049` · **Category:** B Running other programs
- **Evidence:** let editor = p.get("app").and_then(Value::as_str).map(str::to_string).unwrap_or_else(|| app.ui.settings.external_editor.clone());
if let Some(f) = app.services.open_with.as_mut() && let Err(e) = f(&path, &editor) { ... }
// host (apps/lightcraft/src/main.rs:444): std::process::Command::new(app) ... c.arg(path).spawn()
- **Explanation:** A free-form string from command params (or settings) becomes the program the desktop host runs. Any local process that can reach the opt-in, unauthenticated control server (or an MCP agent via `mcp --connect`) can start an arbitrary executable with one argument (the rendered edit copy). This is a design gap, not malicious code, and R1 already logged it as F1. It is referenced here only as the UI-side entry point.
- **Verifier (downgraded, final Low):** The code path is real and reachable. At crates/ui-egui/src/menus.rs:1049, `photo.editInExternal` takes the `app` param, falling back to settings.external_editor (state.rs:148, which the user edits in panels/settings.rs:144). It passes that value to services.open_with. On Linux and Windows, open_with in apps/lightcraft/src/main.rs:426-446 runs `Command::new(app).arg(path).spawn()`. On macOS it runs `open -a app`. The id is reachable from the control channel: control.rs:177 dispatches "engine.execute"/"ui.menu.invoke"/"command" to app.run (lib.rs:418), which calls menus::run_ui_command.

I downgraded the severity because the exposure is small:
(1) The control server is opt-in. It only starts with --control or LIGHTCRAFT_CONTROL_PORT.
(2) It binds only to 127.0.0.1 (control_server.rs:28).
(3) Browser cross-origin attacks are blocked. parse_line rejects any line that is not a JSON object with a `method` and closes the connection, so an HTTP POST's request line kills the connection before the body runs (test at control_server.rs:185-195).
So the only callers are local processes running as the same user, which can already spawn any program themselves, and agents the user deliberately connected via `mcp --connect`. Neither gains new privilege.

Launching a user-chosen external editor is a legitimate Lightroom-parity feature ("Edit in External Editor"). Accepting the editor as a param is consistent with the "everything is a command" design. Nothing here is hidden, obfuscated or malicious. The executable only receives one argument, the rendered edit-copy path, and runs with no shell interpolation on Linux/Windows.

What remains is a defense-in-depth point: there is no authentication or allowlist on the `app` param of an unauthenticated local control channel. That is worth noting, but Low rather than Medium. It duplicates R1 F1.

### 9. [Low] A control-channel client can read the system clipboard via ui.clipboard paste

- **Chunk:** R4-ui-a · **Where:** `crates/ui-egui/src/control.rs:293` · **Category:** D Data collection
- **Evidence:** "ui.clipboard" => { if !ctx.text_edit_focused() { return err(...) }
  match s("action") { ... Some("paste") => match s("text") { Some(text) => app.synthetic.push(egui::Event::Paste(text.to_string())), None => ctx.send_viewport_cmd(egui::ViewportCommand::RequestPaste), }, ...
- **Explanation:** With no `text`, paste asks the host to paste the OS clipboard into the focused field. A control client can click a field (ui.clickWidget), paste, then read the field via ui.inspect or ui.screenshot, which gives it the user's clipboard. copy/cut write field text to the OS clipboard. Only reachable when the user starts the app with the control server enabled. This belongs to the already-logged unauthenticated loopback control server issue and is not malicious.

### 10. [Low] External editor setting is a free-text program executed elsewhere (reachable via the known control channel issue)

- **Chunk:** R5-ui-b · **Where:** `crates/ui-egui/src/panels/settings.rs:144` · **Category:** B - running other programs (indirect)
- **Evidence:** settings.rs:144 `ui.add(egui::TextEdit::singleline(&mut app.ui.settings.external_editor).hint_text(crate::i18n::tr("System default"))...)`; state.rs:148 `pub external_editor: String,` (UiState is documented as readable/settable through the control channel); executed in menus.rs:1049 (other chunk): `let editor = p.get("app").and_then(Value::as_str).map(str::to_string).unwrap_or_else(|| app.ui.settings.external_editor.clone());`
- **Explanation:** This is an ordinary Lightroom-style feature: the user names the program that opens a TIFF copy. A client on the opt-in, unauthenticated loopback control server can choose that program, either through the setting or through the `app` parameter. That follows from the control-server issue already logged in doc.md and is not new malicious code. It is listed here only so the person reviewing menus.rs can check how the program is launched.

### 11. [Low] Unbounded recursion on hierarchical keyword depth from imported XMP (stack overflow)

- **Chunk:** R6-catalog-meta-ai · **Where:** `crates/catalog/src/keywords.rs:284` · **Category:** robustness
- **Evidence:** fn build(at: Option<String>, levels: &HashMap<String, Level>, kids: &mut HashMap<Option<String>, Vec<String>>) -> Vec<KeywordNode> { ... let children = build(Some(l.clone()), levels, kids); ... }  // the depth equals the number of '|' levels in a photo keyword. Photo keywords come from keywords::from_file(&m.keywords, &m.hierarchical_keywords) (engine/src/files.rs:41, engine/src/sidecar.rs:139) with no level cap. MAX_LEVELS = 64 (keywords.rs:57) applies only to parse_keyword_list.
- **Explanation:** A crafted XMP sidecar or embedded XMP with one lr:hierarchicalSubject value like 'a|a|a|...' with hundreds of thousands of levels is a single text node, so it passes the XMP limits (200k nodes, depth 64). keyword_tree() would then recurse once per level and overflow the stack. That aborts the process rather than raising a catchable panic, and it would happen again every time the keyword tree is built while the photo is in the catalog. The same applies to the `learn` recursion at line 499. This is a denial-of-service robustness issue, not malicious behaviour. I found it by reading the code and did not run it.

### 12. [Low] quick-xml 0.38.4 (RUSTSEC-2026-0194, quadratic duplicate-attribute check) is reachable when merging XMP sidecars

- **Chunk:** S14-dependencies · **Where:** `crates/meta/src/xmp_merge.rs:117` · **Category:** vulnerable-dependency
- **Evidence:** Cargo.toml:55 `quick-xml = "0.38"` (locked 0.38.4; patched >= 0.41.0). xmp_merge.rs scan(): `for a in e.attributes() {` (default with_checks(true)), run on the existing sidecar packet; it caps element count (`count > 200_000 || stack.len() > 64`) but not the number of attributes on one tag. By contrast, xmp.rs:231 and gpx.rs:122 use `.attributes().with_checks(false)`.
- **Explanation:** A crafted XMP sidecar with one start tag carrying a huge number of distinct attributes can tie up a CPU core for minutes or longer when LightCraft merges metadata into it on save. This is a denial-of-service bug, not malicious behaviour. Fix: upgrade to quick-xml >= 0.41 or use with_checks(false) here. RUSTSEC-2026-0195 does not apply because only quick_xml::Reader is used (no NsReader).

### 13. [Low] Model-download TLS uses an alpha, unaudited crypto provider (rustls-rustcrypto 0.0.2-alpha), which pulls in advisory-flagged crates

- **Chunk:** S14-dependencies · **Where:** `crates/fetch/Cargo.toml:15` · **Category:** supply-chain
- **Evidence:** `rustls-rustcrypto = "0.0.2-alpha"`; http.rs:173 `ClientConfig::builder_with_provider(Arc::new(rustls_rustcrypto::provider()))...with_root_certificates(roots).with_no_client_auth()`. This brings in rsa 0.9.10 (RUSTSEC-2023-0071, Marvin) and rustls-webpki 0.102.8 (RUSTSEC-2026-0049/0098/0099/0104).
- **Explanation:** None of these advisories can be reached here. No client certificate is used, so RSA is only used to verify server signatures with public keys, while the Marvin attack needs private-key operations. rustls-rustcrypto uses webpki 0.102 only for `webpki::alg_id` constants; certificate chains are checked by rustls 0.23.45 with rustls-webpki 0.103.15, which is patched. The concern is quality: the TLS for model downloads depends on a pre-release crypto provider whose authors do not recommend it for production. The impact is reduced by the per-model SHA-256 pins described in the crate. Not malicious.

### 14. [Info] Web host open_url lacks the https-only check the desktop host has

- **Chunk:** R1-entry-network-tooling · **Where:** `apps/lightcraft-web/src/web.rs:230` · **Category:** A - Network
- **Evidence:** open_url: Some(Box::new(|url: &str| { let w = web_sys::window().ok_or("no window")?; w.open_with_url_and_target(url, "_blank") ... })) vs desktop main.rs:449 `if !url.starts_with("https://") { return Err("only https links are opened".into()); }`
- **Explanation:** Every current caller passes a constant https URL or the OpenStreetMap link built from GPS numbers, so nothing exploitable today. Adding the same https check would cover future callers.

### 15. [Info] Packaging metadata copied from sibling apps does not match LightCraft

- **Chunk:** R1-entry-network-tooling · **Where:** `packaging/windows/lightcraft.wxs:10` · **Category:** K - Code doesn't match stated purpose
- **Evidence:** wxs header: "`.pcraft` owned by LightCraft, and LightCraft offered under \"Open with\" for PSD/PSB" but the MSI only writes HKLM App Paths and Software\LightCraft\InstallFolder; Info.plist.in declares .pcraft/.psb/.qoi UTIs; metainfo keywords psd/layers; nix/package.nix comment "only crates.io, no git sources" vs Cargo.toml craft-ui git dependency
- **Explanation:** These are leftover comments and declarations. The artifacts do less than they say, not more: no file associations or handlers are actually registered. Cosmetic and build-correctness only.

### 16. [Info] xtask probes system RAM / load average (dev tooling only)

- **Chunk:** R1-entry-network-tooling · **Where:** `xtask/src/main.rs:117` · **Category:** D - Data collection
- **Evidence:** available_ram_mb(): reads /proc/meminfo MemAvailable, or powershell `(Get-CimInstance Win32_PerfFormattedData_PerfOS_Memory).AvailableMBytes`, or vm_stat; total_ram_gb(): sysctl -n hw.memsize; bench.rs:54 Command::new("uptime")
- **Explanation:** The RAM figure only sets CARGO_BUILD_JOBS / RUST_TEST_THREADS during `cargo xtask ci`, and the load average is only written to target/bench/history.jsonl. Nothing is sent anywhere and it does not change behaviour on VMs, so it is not sandbox detection. Recorded so the check is on file.

### 17. [Info] photo.duplicate TOCTOU between exists() check and fs::copy

- **Chunk:** R2-engine-a · **Where:** `crates/engine/src/cmd/convert.rs:292` · **Category:** C File destruction/tampering
- **Evidence:** convert.rs:288-292 `while dst.exists() { dst = dir.join(format!("{stem}-copy-{n}{ext}")); n += 1; } std::fs::copy(src, &dst)`
- **Explanation:** If a file with that name appears between the check and the copy, fs::copy overwrites it. This needs a race and is not malicious.

### 18. [Info] Command clients can read local file contents back through import and metadata commands

- **Chunk:** R2-engine-a · **Where:** `crates/engine/src/cmd/query.rs:111` · **Category:** D Data collection
- **Evidence:** query.rs:111-131 photo.allMetadata returns every EXIF/GPS/XMP tag of a catalog photo's file; browse.rs:20-48 library.browse adds any folder's photos as Local records; keywords.rs:104-117 keyword.import reads any UTF-8 file of up to 16 MiB into the keyword list, which keyword.list returns.
- **Explanation:** The data stays local and nothing in this scope transmits it. It is relevant only together with the known unauthenticated control channel or an untrusted MCP client.

### 19. [Info] Runs sysctl / reads /proc/meminfo to learn RAM size

- **Chunk:** R3-engine-b · **Where:** `crates/engine/src/memory.rs:46` · **Category:** system-info
- **Evidence:** std::process::Command::new("/usr/sbin/sysctl").args(["-n", "hw.memsize"]).output()  (also /sbin/sysctl -n hw.physmem at :51, std::fs::read_to_string("/proc/meminfo") at :40)
- **Explanation:** The program path and arguments are fixed. The output is only parsed as a number, and that number only sizes the decode cache (RAM/4, clamped to 256 MiB–1.5 GiB; the LIGHTCRAFT_MEMORY_MB env variable overrides it). The app does nothing different on low-RAM machines apart from that clamp, so this is not sandbox detection.

### 20. [Info] Library prefs.json can move the smart-previews folder elsewhere

- **Chunk:** R3-engine-b · **Where:** `crates/engine/src/library.rs:344` · **Category:** file-tampering
- **Evidence:** self.smart_previews_dir = prefs.smart_previews_dir.filter(|_| on_disk).map(PathBuf::from);  // smart.rs:143-158 migrate() only acts on entries with extension "lcsp"
- **Explanation:** A library's settings file can send smart-preview proxy writes to another folder. Moving or discarding previews (smart::migrate) only touches *.lcsp files, and check_writable never creates missing parent folders. A hostile shared library could at most create or remove LightCraft's own .lcsp proxy files somewhere else; it cannot reach the user's photos.

### 21. [Info] In-app input timestamps pace background AI denoise (benign)

- **Chunk:** R4-ui-a · **Where:** `crates/ui-egui/src/panels/denoise.rs:45` · **Category:** D Data collection
- **Evidence:** let (working, moving) = ctx.input(|i| { let keys = i.events.iter().any(|e| matches!(e, egui::Event::Key { .. } | egui::Event::Text(_) | egui::Event::MouseWheel { .. })); (i.pointer.any_down() || keys, i.pointer.is_moving()) });
if working { app.caches.last_input = now; } ... fn pace_for(focused, minimized, worked, moved, watching) -> "pause" | "light" | "normal" | "full"
- **Explanation:** Only the time of the last key press or pointer move inside the app's own window is kept. Key contents are never recorded. It is used to pause or slow background denoise while the user is working, and nothing is stored or sent. Recorded so other auditors don't re-investigate it.

### 22. [Info] Background face indexing adjusts its pace from idle time and window focus

- **Chunk:** R5-ui-b · **Where:** `crates/ui-egui/src/panels/faces.rs:22` · **Category:** G - hidden triggers (checked, benign)
- **Evidence:** faces.rs:22-48 `fn scan_pace(...)` / `fn pace_for(focused, minimized, worked, moved, watching)` returns "pause"/"light"/"normal"/"full"; faces.rs:720-768 `pump()` calls `app.session.execute("faces.pump", &json!({"pace": pace}))` and returns early unless recognition is active.
- **Explanation:** The indexer works harder when the user is idle, and it only runs after the user turns on face recognition and installs a model. The work is local face indexing of the user's own catalog. Progress shows in Settings and the activity stack, and the UI says everything stays on the computer. It is idle-aware for responsiveness, not to hide anything.

### 23. [Info] Show on Map opens OpenStreetMap with the photo's GPS coordinates when clicked

- **Chunk:** R5-ui-b · **Where:** `crates/ui-egui/src/panels/right.rs:666` · **Category:** A - network (indirect, user-initiated)
- **Evidence:** right.rs:666 `let url = format!("https://www.openstreetmap.org/?mlat={la:.6}&mlon={lo:.6}#map=15/{la:.6}/{lo:.6}");` followed by `crate::links::open(app, &url)`, inside `if text_button(ui, "showOnMap", ...).clicked()`
- **Explanation:** The host is fixed, the button is clearly labelled, and the browser only gets the coordinates when the user clicks it. This is expected behaviour.

### 24. [Info] Local sidebar reads HOME/USERPROFILE and lists folder names under it

- **Chunk:** R5-ui-b · **Where:** `crates/ui-egui/src/panels/left.rs:351` · **Category:** D - data collection (checked, benign)
- **Evidence:** left.rs:351 `let home = std::env::var("HOME").or_else(|_| std::env::var("USERPROFILE")).unwrap_or_default();` then Pictures/Desktop/Downloads/Home rows; left.rs:476-490 `list_subfolders` reads `std::fs::read_dir(path)` for directory names only (dot-dirs skipped), only when the user expands a row, depth capped at 8; left.rs:298-340 `missing_count` runs `Path::exists` on library photo paths only.
- **Explanation:** This copies Lightroom's Local section. It reads directory names and checks whether files exist, nothing more. No file contents are read and nothing is sent or stored outside the app's UI state.

### 25. [Info] Host name, pid and program written to the library folder's lock-owner file

- **Chunk:** R6-catalog-meta-ai · **Where:** `crates/catalog/src/lock.rs:73` · **Category:** data-collection
- **Evidence:** let env = ["COMPUTERNAME", "HOSTNAME"].iter().find_map(|k| std::env::var(k).ok()...); env.or_else(|| std::fs::read_to_string("/etc/hostname").ok())...  // line 115: std::fs::write(dir.join(OWNER), json)
- **Explanation:** This is justified: the data feeds the "library already open in X on host Y" message. It stays local and is never sent anywhere, and the file is removed on clean close. If a library sits on a shared or synced folder, the computer name becomes visible to others with access to that folder.

### 26. [Info] Dev benchmark example wipes the directory passed on its command line

- **Chunk:** R6-catalog-meta-ai · **Where:** `crates/catalog/examples/persist_bench.rs:78` · **Category:** file-destruction
- **Evidence:** let dir = std::path::PathBuf::from(args.next().ok_or("usage: persist_bench <scratch dir> [photos…]")?); ... let _ = std::fs::remove_dir_all(&dir);  (also line 214)
- **Explanation:** The header comment documents that the scratch directory is wiped for each size. It is an example and does not ship in the product, but running it with a real folder as the argument would delete that folder recursively. It is not triggered automatically.

### 27. [Info] meta crate root lacks the panic-deny attribute; a few infallible expect() calls and direct indexing

- **Chunk:** R6-catalog-meta-ai · **Where:** `crates/meta/src/lib.rs:12` · **Category:** coding-standard
- **Evidence:** #![forbid(unsafe_code)] with no #![deny(clippy::unwrap_used, clippy::expect_used, ...)] (every other crate root in the chunk has it). xmp.rs parse_tree: `stack.pop().expect("len > 1")`, `.expect("root")`. Also catalog/src/query.rs date_groups `&d[..4]`, catalog/src/model.rs Stack::top `self.photos[0]`.
- **Explanation:** These depart from the project's never-crash rules. The xmp.rs expect() calls cannot fail by construction. The byte-slicing and indexing could panic only on a hand-crafted catalog snapshot. None of this is malicious.

### 28. [Info] Embedded XOR keystream seed for Sony SR2SubIFD decryption (justified)

- **Chunk:** R7-raw-tiff · **Where:** `crates/raw/src/vendor/sr2.rs:52` · **Category:** H-obfuscation
- **Evidence:** const KEY: [u8; 4] = [0x11, 0x22, 0x33, 0x44]; ... const SEED: [u32; 127] = [0x54c2c4d6, ...]; fn apply_keystream(block: &mut [u8]) { ... ring[i] ^= ring[(i + 64) % SEED.len()]; ... *b ^= k; }
- **Explanation:** A 127-word seed plus the recurrence k[n]=k[n-63]^k[n-127] decrypts the encrypted SR2SubIFD of the ARW the user opened. The input is capped at MAX_LEN = 1 MiB and the key must be 11 22 33 44. The only use of the result is SubIfd::shorts() / short_bytes(), which read the black-level (0x7310) and WB_RGGBLevels (0x7313) SHORT tags. Nothing embedded in the binary gets decrypted or executed, and the structure matches the publicly known Sony SR2 cipher. Recorded only because the H checklist asks about XOR loops.

### 29. [Info] Sony maker-note byte-substitution decipher table (justified)

- **Chunk:** R7-raw-tiff · **Where:** `crates/raw/src/vendor/arw.rs:89` · **Category:** H-obfuscation
- **Evidence:** const DECIPHER: [u8; 256] = { let mut inv = [0u8; 256]; let mut p = 0usize; while p < 256 { let c = if p < 249 { p * p % 249 * p % 249 } else { p }; inv[c] = p as u8; p += 1; } inv };
- **Explanation:** This is the const-evaluated inverse of Sony's documented p^3 mod 249 substitution. In tag2010_wb() it is applied to 6 bytes of the maker-note Tag2010 block, at a model-specific offset, to read a fallback white balance. The function has no other effects.

### 30. [Info] Literal NUL/control bytes inside test byte-string literals make sources look binary

- **Chunk:** R7-raw-tiff · **Where:** `crates/raw/src/vendor/cr3.rs:446` · **Category:** review-hygiene
- **Evidence:** cr3.rs:446 `let mut tiff = b"II*^@^H^@^@^@^A^@".to_vec();` (raw 0x00/0x08/0x01 bytes); pef.rs:448 `let mut note = b"AOC^@MM".to_vec();` — `file` reports both .rs files as "data" and grep prints "binary file matches".
- **Explanation:** Both lines are harmless test fixtures: a TIFF header (II*\0\x08\0\0\0\x01\0) and the Pentax AOC\0 maker-note magic. Because the bytes are raw rather than escaped, grep-based review tools treat these files as binary and can skip them unless run with -a. I suggest writing them as \0 / \x08 escapes. They have no runtime effect.

### 31. [Info] Only unsafe in chunk: libc clock_gettime in a dev-only benchmark example

- **Chunk:** R8-compute · **Where:** `crates/raster/examples/blur_bench.rs:14` · **Category:** E Low-level/OS calls
- **Evidence:** #[allow(unsafe_code)]
fn cpu_ms() -> f64 { ... let mut ts = libc::timespec { tv_sec: 0, tv_nsec: 0 };
// SAFETY: `clock_gettime` only writes into the timespec we pass.
unsafe { libc::clock_gettime(libc::CLOCK_PROCESS_CPUTIME_ID, &mut ts) };
- **Explanation:** This reads process CPU time for a micro-benchmark. libc is a dev-dependency in crates/raster/Cargo.toml and examples don't ship, so it's benign. It's outside the project's own rule that unsafe lives only in crates/sysmem, but it isn't a security concern.

### 32. [Info] GPU-init crash sentinel writes/removes a marker file at a caller-supplied path

- **Chunk:** R8-compute · **Where:** `crates/gpu/src/backend.rs:163` · **Category:** C File tampering (checked, justified)
- **Evidence:** let _ = std::fs::write(m, format!("GPU device creation started (backends {backends:?}, unix time {secs})\n"));
let r = f();
if let Some(m) = &marker { let _ = std::fs::remove_file(m); }
// caller: apps/lightcraft/src/main.rs:107-111 config_dir().map(|d| d.join("gpu-init.marker")), skipped when LIGHTCRAFT_NO_PREFS is set
- **Explanation:** This is the only production file I/O in the chunk. The file is written in the app's own config folder before the GPU device is created and removed afterwards. It records the backend set and a timestamp, so the next launch can tell that the last run crashed inside the driver. The content is not sensitive and nothing is sent anywhere. Justified.

### 33. [Info] GPU adapter selection skips software (CPU) adapters; adapter info is used locally only

- **Chunk:** R8-compute · **Where:** `crates/gpu/src/ctx.rs:293` · **Category:** L Sandbox detection (checked, justified)
- **Evidence:** if info.device_type == wgpu::DeviceType::Cpu {
    log::info!("gpu: only a software adapter ({}), using the CPU pipeline", info.name);
    return Err(format!("software adapter ({}) skipped: the CPU pipeline is faster", info.name));
}
- **Explanation:** This skips software rasterisers such as llvmpipe and falls back to the CPU pipeline, which is the reference and gives the same output. The same check is in nn.rs:108. The program's behaviour does not change, so this is not evasion. Adapter names and limits go only to log messages and the local perf/fallback strings. No fingerprint is sent or stored.

### 34. [Info] Library lock file records the computer's hostname

- **Chunk:** S1-sweeps-binaries · **Where:** `crates/catalog/src/lock.rs:73` · **Category:** data-collection
- **Evidence:** let env = ["COMPUTERNAME", "HOSTNAME"].iter().find_map(|k| std::env::var(k).ok().filter(|h| !h.trim().is_empty()));
    env.or_else(|| std::fs::read_to_string("/etc/hostname").ok())...
- **Explanation:** The hostname is saved in the lock file inside the library folder so the app can say 'open in LightCraft on <host>'. It stays local and is never sent anywhere, but people who share a library on a network drive will see it. This is justified, ordinary behaviour, listed for completeness.

### 35. [Info] Windows release binary still contains the known cmd /C start file-open path

- **Chunk:** S1-sweeps-binaries · **Where:** `apps/lightcraft/src/main.rs:437` · **Category:** process-execution
- **Evidence:** let mut c = std::process::Command::new("cmd");
                    c.args(["/C", "start", ""]);  (v0.6.0 lightcraft.exe imports CreateProcessW; its strings include 'explorer', 'rundll32 url.dll,FileProtocolHandler' for URLs and 'cmd' for files)
- **Explanation:** This is already logged in doc.md as a Medium bug (command injection through a file name containing '&'). The release binary confirms the shipped code matches the source: URLs open through rundll32 url.dll,FileProtocolHandler, while files still go through cmd /C start. This is a reference only, not a new issue.

### 36. [Info] Unmaintained dependencies: paste 1.0.15 and ttf-parser 0.25.1

- **Chunk:** S14-dependencies · **Where:** `Cargo.lock` · **Category:** unmaintained-dependency
- **Evidence:** RUSTSEC-2024-0436 (paste, a proc-macro used via faer-core/gemm/candle) and RUSTSEC-2026-0192 (ttf-parser, via ab_glyph -> lightcraft-engine). Both are informational = "unmaintained" with no patched version.
- **Explanation:** These are flagged as unmaintained, not as vulnerable. paste only runs at compile time. ttf-parser parses fonts that are bundled or come from craft-fonts.

### 37. [Info] Dependencies ship prebuilt binary objects (standard upstream practice)

- **Chunk:** S14-dependencies · **Where:** `Cargo.lock` · **Category:** supply-chain
- **Evidence:** ring-0.17.14/pregenerated/*-nasm.o (Windows asm objects, used when the crate has no .git); wit-bindgen-0.57.1 build.rs copies src/rt/libwit_bindgen_cabi.a into OUT_DIR (wasm targets only); windows_*-0.52.6/0.53.1 build.rs: `cargo:rustc-link-search=native={CARGO_MANIFEST_DIR}/lib` (import libraries).
- **Explanation:** These are opaque binaries that get linked into the build. Their .crate checksums match crates.io and Cargo.lock, and all are well-known upstream packaging choices. Noted for completeness only.

### 38. [Info] Environment / container detection in a dev-only dependency (is-docker, is-wsl via open)

- **Chunk:** S14-dependencies · **Where:** `Cargo.lock` · **Category:** environment-detection
- **Evidence:** is-docker-0.2.0/src/lib.rs:7 `fs::metadata("/.dockerenv")`, :11 `fs::read_to_string("/proc/self/cgroup")`; is-wsl-0.4.0/src/lib.rs:30 `read_to_string("/proc/version")`. Dependency path: open 5.4.4 <- egui_kittest (dev-dependency of lightcraft-ui-egui only).
- **Explanation:** These checks match checklist item L, but they are legitimate: the `open` crate uses them to pick a URL launcher under WSL or Docker. They are pulled in only through a test dependency, so they are not built into the app or CLI. Not a sandbox-evasion pattern.

### 39. [Info] Dynamic run: no outbound network, no child processes, no bait access, no VM checks (behaviour matched expectations)

- **Chunk:** S15-dynamic-run · **Where:** `audit/S15-dynamic-run.md` · **Category:** dynamic-analysis
- **Evidence:** 7 app traces (a_render 260 lines, b_snapshot 8461, b_mcp 64, c_app1 9507, c_app2 3531, d_render 284, d_app 6730): bait matches = 0 in every trace. The only AF_INET calls are `bind(12, {sa_family=AF_INET, sin_port=htons(18615), sin_addr=inet_addr("127.0.0.1")}, 16) = 0` plus accept4 from 127.0.0.1, and the same on 18616 in (d). execve after the app's own exec: 0. Syscall histogram for c_app1: `8763 22:12` / `744 22:23`, i.e. nothing between 22:12:47.365 and 22:23:17.451 (the idle period). VM paths (/proc/cpuinfo, /sys/class/dmi, /proc/1/*, /proc/uptime, mountinfo, osrelease, /proc/version, /run/systemd, container_info.json) appear in no app trace.
- **Explanation:** The disguised sandbox gave a malware-style detector the cues it looks for, and a reason to wait (10.5 min idle, clock +500 days). The app never touched the bait, never probed the VM-related files, never tried to reach the network, and wrote only to its library, config, the export targets I chose, and Mesa's cache. Original photos were never opened with write flags and their sha256 equals the docs/images sources. Limits: cpuid, uname, mountinfo and TracerPid could not be hidden (cpuid is covered by S1's static scan); Linux only; software GL.

### 40. [Info] Camera-card detection lists /media and /mnt and stats <mount>/DCIM in a background thread

- **Chunk:** S15-dynamic-run · **Where:** `crates/engine/src/devices.rs:31` · **Category:** data-collection
- **Evidence:** devices.rs:42-46: `out.push(format!("/media/{user}").into()); out.push(format!("/run/media/{user}").into()); ... out.push("/media".into()); out.push("/mnt".into());` Trace b_snapshot: `openat(AT_FDCWD, "/mnt", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3`, `statx(AT_FDCWD, "/mnt/attach/DCIM", ...) = -1 ENOENT`, repeated for every child of /mnt. Rescan runs on thread "lightcraft-devices" when the cached list is more than 2 s old (MAX_AGE).
- **Explanation:** This reads directories the user didn't open, but only lists the mount roots and checks whether a DCIM folder exists, for File > Import from Device. No file contents are read. It's justified and matches its stated purpose; recorded so readers know where the /media and /mnt accesses in the traces come from.

### 41. [Info] Library lock owner file records the machine's host name

- **Chunk:** S15-dynamic-run · **Where:** `audit/S15-dynamic-run.md` · **Category:** data-collection
- **Evidence:** c_app1.strace: `openat(AT_FDCWD, "/etc/hostname", O_RDONLY|O_CLOEXEC) = 12` immediately followed by `write(12, "{\n  \"pid\": 142,\n  \"host\": \"alex-laptop\",\n  \"program\": \"LightCraft\",\n  \"version\": \"0.6.0\",\n  \"since\": 1791843145\n}", 113)` to `~/Pictures/LightCraft Library/catalog.lock.owner`. The file is unlinked on quit.
- **Explanation:** The host name is read only to label the library lock, so another machine can tell who holds a shared or external library. It stays inside the library folder and is never sent anywhere (no network). Justified.

### 42. [Info] Startup-failure path runs zenity for an error dialog; libX11 falls back to TCP 127.0.0.1:6000

- **Chunk:** S15-dynamic-run · **Where:** `apps/lightcraft/src/main.rs` · **Category:** process-execution
- **Evidence:** out/failed1/c_app2.strace: `execve("/usr/bin/zenity", ["zenity", "--no-markup", "--error", "--title", "LightCraft could not start", "--text", "LightCraft could not open its window: winit EventLoopError: ...` (ENOENT along PATH) and `connect(4, {sa_family=AF_INET, sin_port=htons(6000), sin_addr=inet_addr("127.0.0.1")}, 16) = -1 ECONNREFUSED`.
- **Explanation:** The only external programs the app tried to start in any run. They come from the rfd message-dialog backend after the window failed to open; the TCP connect is libX11's standard fallback for DISPLAY=:0 when the unix socket is missing. Both are expected and loopback/local only.
