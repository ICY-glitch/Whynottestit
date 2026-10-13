# R1 entry/network/tooling audit notes

Scope: files listed in audit/chunks/R1-entry-network-tooling.txt (110 files, ~20.7k lines).
Question: does any code behave like malware? Checklist A-L applied per file.
Tiers: 1 = touches A-L capabilities (every line read); 2 = ordinary logic (every line read); 3 = pure computation (proved no A-L, skimmed).

| path | lines | tier | capabilities | status | note |
|---|---|---|---|---|---|
| .cargo/config.toml | 2 | 3 | J (alias only) | done | `xtask` cargo alias only; no build hooks/runner overrides |
| rustfmt.toml | 2 | 3 | - | done | formatting only |
| clippy.toml | 4 | 3 | - | done | test-only lint relaxations |
| omastore.toml | 11 | 3 | - | done | store manifest, asset names only |
| packaging/linux/ai.storyteller.lightcraft.mime.xml | 3 | 3 | - | done | empty mime-info |
| packaging/macos/entitlements.plist | 9 | 3 | I | done | empty entitlements dict (no extra privileges) |
| packaging/web/_headers | 9 | 3 | - | done | cache/nosniff headers (no CSP: known issue) |
| packaging/web/.htaccess | 15 | 3 | - | done | mime/deflate/cache only |
| packaging/linux/...desktop | 12 | 3 | - | done | standard desktop entry, Exec=lightcraft %F |
| packaging/linux/postinst.sh | 13 | 1 | B | done | refreshes desktop/mime/icon caches only; nothing else |
| apps/lightcraft-web/worker.js | 15 | 2 | - | done | inits wasm module in worker; no network/eval |
| apps/lightcraft-cli/src/alloc_release.rs | 12 | 2 | E (indirect) | done | calls sysmem::release_free_memory |
| apps/lightcraft/src/alloc_release.rs | 12 | 2 | E (indirect) | done | identical |
| crates/sysmem/Cargo.toml | 18 | 3 | E | done | no deps |
| crates/sysmem/src/lib.rs | 80 | 1 | E | done | single extern "C" malloc_zone_pressure_relief(NULL,0) on macOS only; matches docs; no other unsafe |
| crates/mcp/Cargo.toml | 16 | 3 | - | done | deps internal + serde_json/log |
| crates/fetch/Cargo.toml | 17 | 3 | A | done | rustls + rustls-rustcrypto (alpha) + webpki-roots + sha2 |
| xtask/Cargo.toml | 17 | 3 | - | done | serde_json, sha2, flate2, brotli |
| crates/segment/Cargo.toml | 25 | 3 | - | done | candle, fetch |
| crates/segment/src/bin/lightcraft-sam3-worker.rs | 18 | 1 | A | done | explicit binary: serve on addr from argv (default 127.0.0.1:8793) |
| crates/segment/tests/remote.rs | 21 | test | - | done | frame validation tests |
| crates/segment/examples/sam3_bench.rs | 25 | test | - | done | timing with synthetic image |
| apps/lightcraft/build.rs | 28 | 1 | J | done | winresource icon/version embedding on Windows only; no network/exec beyond rc compiler via crate |
| crates/engine/build.rs | 99 | 1 | J,H(include_bytes) | done | reads local CRAFT_FONTS_DIR manifest (env opt-in) and emits include_bytes! of listed font files; writes only OUT_DIR; no network/exec. Manifest paths are not confined to the dir (build-time, developer-supplied input) - not a finding |
| crates/ui-egui/build.rs | 208 | 1 | J | done | generates tr_format! macro from locales/*.json and credits tables from contributors/*.json/toml; writes OUT_DIR only; strings escaped via serde_json/{:?} |
| Cargo.toml | 153 | 3 | - | done | workspace deps; craft-ui git dep pinned to rev 5c874d5; libc dev-only; dhat optional; panic unwind native, abort web |
| apps/lightcraft-cli/Cargo.toml | 38 | 3 | - | done | internal deps, optional dhat/heif |
| apps/lightcraft/Cargo.toml | 39 | 3 | - | done | eframe, rfd, muda (macOS), winresource build-dep |
| apps/lightcraft-web/Cargo.toml | 79 | 3 | - | done | web-sys features: no fetch/XHR/WebSocket/Clipboard/Geolocation; FS Access + IndexedDB + Worker only |
| apps/lightcraft/src/main.rs | 1158 | 1 | B,C,D(env),F | done | Startup traced: logger install -> panic hook (append to temp/lightcraft-panics.log locally) -> alloc hook -> args -> log rotate in config/logs -> read ui.json -> GPU marker -> eframe; in creator: open library, set SAM3 dir/mirrors path (no download), control server ONLY if --control/LIGHTCRAFT_CONTROL_PORT. No network, no extra threads at launch beyond eframe. Process spawns: open/xdg-open/rundll32 (https-only), explorer /select (quoted), open_with (`cmd /C start` known issue; `Command::new(app)` with user/`app` param, see finding F1). File writes: ui.json atomic, corrupt ui.json renamed aside, gpu marker. display_profile_dir only sets a dialog start dir. |
| apps/lightcraft/src/control_server.rs | 267 | 1 | A,F(thread on opt-in) | done | binds 127.0.0.1 only, line-bounded, max 16 conns, rejects HTTP lines; no auth (known issue). Thread only when port given. |
| apps/lightcraft/src/logging.rs | 689 | 1 | C,D(env) | done | log rotation in config/logs (3 files), 16 MB cap, panic hook adds to file; reads LIGHTCRAFT_LOG/RUST_LOG only |
| apps/lightcraft/src/dialog_filter.rs | 145 | 3 | - | done | rfd extension glob helpers; include_str! of own sources in test |
| apps/lightcraft/src/native_menu.rs | 526 | 2 | E(indirect via muda) | done | macOS menu via muda; no direct FFI; keyboard events only via menu accelerators (no keylogging) |
| apps/lightcraft/examples/gen_photos.rs | 45 | test | C (user OUT_DIR) | done | writes procedural JPEGs to argv dir |
| apps/lightcraft-cli/src/main.rs | 899 | 1 | A(user --connect),C(user paths),D(env) | done | Subcommands only on explicit argv. Startup: alloc hook + stderr logger, nothing else. mcp/run --connect connects to user-given addr (default 127.0.0.1:7980). render refuses overwriting the input/sidecar (original_guard). calibrate writes profiles to config/camera-profiles or --out. Snapshot Services have no open_with/open_url. No network/exec of its own. |
| apps/lightcraft-cli/tests/cli.rs | 579 | test | B(own binary),C(temp) | done | spawns built CLI only; deletes under temp_dir/lightcraft-cli-test-<pid> only |
| crates/fetch/src/lib.rs | 467 | 1 | A,C,D(env token) | done | Model downloader. Writes only `<dir>/<name>.part` then rename; filename validated (no separators, no reserved names). Deletes only its own .part on mismatch. Bearer token from caller-named env var sent only over https to the configured mirror host (never to redirect hosts). Redirects may go https->http (integrity relies on SHA-256 pin; unpinned files are size-capped only) - see finding. No hardcoded hosts. |
| crates/fetch/src/http.rs | 550 | 1 | A | done | Minimal HTTP/1.1 GET over std TcpStream + rustls/RustCrypto + webpki-roots; no proxy, no cert override; header injection guarded; head size caps. Only GET; no upload/body. User-Agent LightCraft/<ver> only. |
| crates/fetch/src/tests.rs | 388 | test | A(loopback) | done | local 127.0.0.1 servers only; deletes in temp only |
| crates/segment/src/fetch/mod.rs | 79 | 1 | A(config),D(env) | done | DEFAULT_MIRRORS is EMPTY; mirrors only from LIGHTCRAFT_SAM3_MIRRORS / user mirrors file. weights pinned (size+SHA-256); vocab.json/merges.txt unpinned (16 MB cap, parsed only). Token env LIGHTCRAFT_SAM3_TOKEN. |
| crates/segment/src/remote.rs | 207 | 1 | A,D(env) | done | Client connects only to loopback IP (LIGHTCRAFT_SAM3_REMOTE), sends RGB + prompt, frames bounded. serve(): loopback-only bind, Metal-only, catch_unwind per client. No auth (loopback/SSH-tunnel design; opt-in explicit binary). |
| crates/segment/src/lib.rs | 265 | 2 | - | done | model load/encode/segment; no I/O beyond weights/tokenizer reads |
| crates/segment/src/weights.rs | 151 | 2 | C(read only) | done | safetensors positional reads, header bounded 100 MB, offsets checked vs file length; tests write temp |
| crates/segment/src/tokenizer.rs | 177 | 2 | - | done | reads vocab.json/merges.txt from model dir; BPE only |
| crates/segment/src/clip.rs | 81 | 3 | - | done | candle math |
| crates/segment/src/neck.rs | 77 | 3 | - | done | candle math |
| crates/segment/src/mask.rs | 130 | 3 | - | done | resampling math, bounds-checked |
| crates/segment/src/detector.rs | 308 | 3 | - | done | only candle imports; math |
| crates/segment/src/nn.rs | 242 | 3 | - | done | only candle imports; math |
| crates/segment/src/tracker.rs | 253 | 3 | - | done | only candle imports; math |
| crates/segment/src/vit.rs | 170 | 3 | - | done | only candle imports; math |
| crates/segment/tests/reference.rs | 97 | test | D(env) | done | runs only with LIGHTCRAFT_SAM3_DIR/REF env set; reads files |
| crates/mcp/src/lib.rs | 34 | 3 | - | done | module wiring; DEFAULT_ADDR 127.0.0.1:7980 |
| crates/mcp/src/backend.rs | 117 | 1 | A | done | Remote: TCP to user-given addr (control channel client), JSON lines only |
| crates/mcp/src/base64.rs | 60 | 3 | H(checked) | done | plain RFC4648 encoder/decoder for MCP image content; decode only used by e2e test; no embedded blobs |
| crates/mcp/src/headless.rs | 251 | 2 | C(user paths) | done | in-process session; export/render write via export::write_file with check_write_target; expand_paths bounded walk |
| crates/mcp/src/server.rs | 262 | 2 | - | done | JSON-RPC over stdio; catch_unwind guard. `"2026-07-28"` is a protocol-version string compare on client _meta, not a clock trigger |
| crates/mcp/src/progress.rs | 165 | 2 | F(scoped threads while serving) | done | stdin/stdout transport threads (scoped), progress notifications |
| crates/mcp/src/tools.rs | 780 | 1 | C(temp png, user path),D(clipboard via UI tool) | done | MCP tools forward to backend; temp render PNGs in temp_dir removed after read; save paths checked by export.checkTarget; `clipboard` tool only operates on focused text field via ui.clipboard (agent-initiated). Every command reachable (incl. photo.editInExternal with `app` when --connect: see finding F1) |
| crates/mcp/src/tests.rs | 401 | test | C(temp) | done | temp dirs only |
| crates/mcp/tests/e2e.rs | 267 | test | A(loopback),C(temp) | done | loopback listeners, temp dirs |
| apps/lightcraft-web/index.html | 49 | 1 | A(same-origin fetch) | done | fetches only ./lightcraft_zh_hans.otf and ./lightcraft_web_bg.wasm (relative, same origin); exposes globalThis.lightcraft.command for page-level automation |
| apps/lightcraft-web/src/lib.rs | 32 | 3 | - | done | module wiring |
| apps/lightcraft-web/src/web.rs | 705 | 1 | C(browser storage),D(URL query) | done | OPFS/IDB storage only; ?reset clears storage only after confirm(); orphan thumbnail cleanup in own store; export = Blob download; open_url = window.open(_blank) (no https check unlike desktop; all callers pass constant/https URLs - Info). No network calls. |
| apps/lightcraft-web/src/safety.rs | 275 | 1 | C(browser storage) | done | Web Locks one-tab guard, panic page, backup zip download, restore into new folder (never deletes) |
| apps/lightcraft-web/src/backend.rs | 345 | 1 | C(browser storage) | done | OPFS/IndexedDB wrapper; clear() only via ?reset path |
| apps/lightcraft-web/src/backup.rs | 338 | 2 | - | done | stored-zip writer/reader; restore keys sanitized (library files whitelist, hash alnum) |
| apps/lightcraft-web/src/files.rs | 313 | 2 | - | done | in-memory mirror of library files |
| apps/lightcraft-web/src/store.rs | 247 | 2 | - | done | content-hashed originals cache, 768 MB budget |
| apps/lightcraft-web/src/workers.rs | 506 | 1 | F(workers at start) | done | spawns same-origin ./worker.js module workers for rendering; messages are render jobs/results only |
| apps/lightcraft-web/src/wire.rs | 524 | 3 | - | done | render job serialization, thumb index; engine imports only |
| apps/lightcraft-web/src/bench.rs | 143 | 3 | - | done | ?bench timing, logs to console |
| xtask/src/main.rs | 929 | 1 | A(curl, opt-in),B,C,D(RAM info) | done | dev tooling only. `ci` reads /proc/meminfo or runs powershell Get-CimInstance / vm_stat / sysctl hw.memsize solely to size CARGO_BUILD_JOBS (not VM detection). `corpus --download` curls ~100 raw.pixls.us CC0 files into corpus/raw, each SHA-256 pinned; mismatches deleted (own files only). |
| xtask/src/web.rs | 328 | 1 | A(127.0.0.1 dev server),B,C | done | runs cargo/wasm-bindgen/wasm-opt; serve_dir binds 127.0.0.1 only, path traversal guarded (.., backslash); removes stale font files in its own output dir |
| xtask/src/bench.rs | 160 | 1 | B,D(uptime) | done | runs `uptime` for load average + git rev, appends target/bench/history.jsonl |
| xtask/src/assets.rs | 136 | 1 | B(git) | done | git ls-files; asset attribution check |
| xtask/src/version.rs | 152 | 1 | C,B | done | `version set` rewrites root Cargo.toml atomically; cargo update |
| xtask/src/parity.rs | 460 | 1 | B,C | done | runs lightcraft-cli commands/controls --json; --write updates docs/parity-checklist.md |
| xtask/src/stats.rs | 118 | 2 | B(cargo) | done | line/test counts |
| xtask/src/ico.rs | 104 | 3 | C(user out) | done | ICO packer |
| xtask/src/layers.rs | 315 | 3 | - | done | dependency layer check from cargo metadata |
| .github/workflows/freebsd.yml | 55 | 1 | A,B,J | done | actions SHA-pinned; craft-fonts pinned by commit; `curl sh.rustup.rs | sh` inside VM (known unpinned-tool issue); permissions contents:read |
| .github/workflows/packaging-lint.yml | 73 | 1 | A,B | done | actionlint tarball by version, no hash (known issue); lint only |
| .github/workflows/windows-arm64.yml | 95 | 1 | B | done | package unsigned, install/uninstall MSI on runner; no secrets |
| .github/workflows/release.yml | 491 | 1 | A,B,J | done | secrets scoped to `release` environment jobs (macOS/Windows/release); version input validated by regex before use and passed via env; nfpm .deb downloaded by version w/o hash (known); riscv64 apt from http ports.ubuntu.com (apt-signed); draft release only; SHA256SUMS generated. No exfiltration of secrets observed. |
| packaging/env.sh | 72 | 1 | B | done | version/env helpers, copy licences, sha256 |
| packaging/macos/import-cert.sh | 50 | 1 | I(temp keychain, CI) | done | imports Developer ID cert into a temporary keychain; deletes p12; writes identity to GITHUB_ENV only. No exfil. |
| packaging/macos/package.sh | 170 | 1 | B | done | build, lipo, codesign hardened runtime w/ empty entitlements, notarize via xcrun notarytool (Apple only), DMG |
| packaging/windows/package.ps1 | 172 | 1 | B,C(staging) | done | cargo build, PE header checks, generated BMPs, wix build, zip |
| packaging/windows/sign.ps1 | 95 | 1 | A,B | done | signtool with pfx (temp dir removed) or Azure Trusted Signing: downloads UNPINNED latest Microsoft.Trusted.Signing.Client from nuget.org and loads its dlib with Azure secrets in env (known unpinned-download category, but this one handles signing creds - Low) |
| packaging/windows/lightcraft.wxs | 184 | 1 | I(registry) | done | per-machine MSI: HKLM App Paths + Software\LightCraft\InstallFolder only; no Run keys/services; launch-on-finish via WixUnelevatedShellExec (UI only, unelevated). Header comment mentions .pcraft/PSD associations that are NOT present (stale comment, harmless) |
| packaging/linux/package.sh | 148 | 1 | A,B | done | FHS stage, nfpm deb/rpm, AppImage via appimagetool `continuous` download (unpinned, known category); update info gh-releases-zsync (consumed only by external AppImageUpdate, app itself has no updater) |
| packaging/linux/nfpm.yaml | 71 | 3 | - | done | deb/rpm metadata; postinst/postremove = postinst.sh |
| packaging/linux/flatpak-bundle.sh | 72 | 1 | A(flathub),B | done | builds bundle from tarball, installs per-user for smoke test |
| packaging/linux/flatpak/ai.storyteller.lightcraft.bundle.yml | 50 | 3 | - | done | finish-args: wayland/x11/ipc/dri/xdg-pictures; NO network, NO home |
| packaging/linux/flatpak/ai.storyteller.lightcraft.yml | 69 | 3 | - | done | same finish-args; offline cargo build |
| packaging/linux/ai.storyteller.lightcraft.metainfo.xml.in | 61 | 3 | - | done | AppStream; keywords/screenshot caption copied from a sibling app (psd/layers) - cosmetic |
| packaging/freebsd/package.sh | 100 | 1 | B | done | build/stage/tar; dry-run stub binaries only echo |
| packaging/icons.sh | 59 | 1 | B | done | resvg/iconutil/xtask ico; temp dir cleanup |
| packaging/web/package.sh | 38 | 1 | B | done | xtask web + zip |
| packaging/macos/Info.plist.in | 110 | 3 | - | done | bundle metadata; declares .pcraft/psb/qoi UTIs (copied from sibling; no handlers) |
| packaging/macos/dmg/generate.py | 149 | 1 | B(resvg) | done | dev script generating DMG background/DS_Store deterministically; writes only in its own dir |
| packaging/linux/ai.storyteller.lightcraft.desktop | 12 | 3 | - | done | (full path for the row above) |
| flake.nix | 115 | 1 | J | done | nixpkgs unstable (unpinned by design, flake.lock pins), craft-fonts pinned by commit; dev shell env only |
| nix/package.nix | 212 | 1 | J | done | buildRustPackage; wraps binaries with LD_LIBRARY_PATH for dlopen'd libs. Comment claims "no git sources" but Cargo.toml has a git dep (craft-ui) - would need outputHashes; build correctness only |
| tools/install-sam3.sh | 130 | 1 | A,D(HF token file),C | done | dev script: reads HF_TOKEN or ~/.cache/huggingface/token, sends only to https://huggingface.co (curl -L does not forward custom auth header cross-host); SHA-256 pinned weights; deletes only its own .part / mismatched model file in model dir |
| tools/install-sam3.ps1 | 60 | 1 | A,D(HF token file),C | done | Windows twin of the above; Invoke-WebRequest to huggingface.co |
| tools/sam3_reference.py | 71 | 2 | - | done | dev script: runs HF transformers locally on a given photo, writes safetensors |
| tools/spectral_camera_table.py | 122 | 2 | B(git rev-parse) | done | dev script: fits matrices from a local rawtoaces-data checkout, writes a .rs file |
| docs/raw/coverage.py | 172 | 2 | C(own tsv) | done | stdlib only; rewrites docs/raw/coverage.tsv |
| docs/showcase/run.py | 20 | 1 | A(127.0.0.1:7980) | done | drives the local control channel from a JSONL file |

## Startup trace (desktop app, before any user action)
logging::install (reads LIGHTCRAFT_LOG/RUST_LOG) -> panic hook (append-only `lightcraft-panics.log` in temp or LIGHTCRAFT_CONFIG_DIR, only on panic) -> logger panic chain -> alloc release hook (macOS malloc_zone_pressure_relief) -> argv parse -> log rotation in <config>/logs -> launch-path resolution (refuses drive roots / home) -> read <config>/ui.json (corrupt file renamed aside) -> GPU init marker -> eframe. In the creator: open library (default ~/Pictures/LightCraft Library), set SAM 3 dir / mirrors-file path (no download), control server ONLY when --control or LIGHTCRAFT_CONTROL_PORT is given, import of argv paths on the import worker, muda menu on macOS. No network, no process spawn, no persistence, no timers beyond the 3 s ui.json save check. CLI: alloc hook + stderr logger only.

## Network paths (all traced end to end)
1. Model downloads: crates/fetch (pure-Rust HTTP/1.1 + rustls). SAM 3 DEFAULT_MIRRORS is empty: only user-configured mirrors (env/mirrors file). Face/denoise downloads go through engine/model_download.rs & face_download.rs (other chunk) with pinned size+SHA-256. No hard-coded hosts in fetch.
2. Control server: 127.0.0.1 only, opt-in (known issue).
3. SAM 3 remote client/worker: loopback-only both ends; separate explicit binary.
4. MCP Remote backend: TCP to user-given address.
5. Web: only same-origin fetches of its own wasm/font; no fetch/XHR/WebSocket APIs in web-sys features.
6. xtask: corpus curl (SHA-256 pinned, opt-in flag), 127.0.0.1 dev server.
7. CI/packaging: GitHub/Apple/Azure/nuget/flathub/rustup downloads during builds only.
No telemetry, no update check, no beaconing anywhere in scope.

## Process spawns (product)
open/xdg-open/rundll32 url.dll (https-only URLs), explorer /select (quoted), `open -a`/`cmd /C start` (known) / `Command::new(app)` for Edit in External Editor (F1). nothing else in scope (engine memory.rs runs sysctl for RAM size: other chunk).

## Findings
- F1 (Medium, new): `photo.editInExternal` accepts a free-form `app` param that becomes `Command::new(app).arg(path).spawn()` (apps/lightcraft/src/main.rs:426-446, crates/ui-egui/src/menus.rs:1049). Reachable through the opt-in unauthenticated control channel (`engine.execute`) and therefore `lightcraft-cli mcp --connect` / `run --connect`: any local process (or an MCP agent) can launch an arbitrary executable with one argument. Escalates the known "unauthenticated control server" issue from app-command access to arbitrary program execution. Not malicious; design gap.
- F2 (Low): crates/fetch follows redirects from https to http (`Url::join` accepts http://, lib.rs:353-360). Integrity holds for SHA-256-pinned files; SAM 3 vocab.json/merges.txt are unpinned (size-capped, parse-only) so a network attacker could tamper with them if a mirror redirects to http. Related to the known "http mirrors allowed" item.
- F3 (Low): packaging/windows/sign.ps1:249 downloads the latest (unversioned, unhashed) Microsoft.Trusted.Signing.Client from nuget.org and loads its dlib into signtool while Azure signing secrets are in the environment. Same class as known "unpinned CI tool downloads", but on the signing path. Also appimagetool `continuous` (packaging/linux/package.sh:113) gets embedded into shipped AppImages.
- F4 (Info): web open_url (apps/lightcraft-web/src/web.rs:230-233) has no https check, unlike desktop; all current callers pass constant/https URLs.
- F5 (Info): metadata copied from sibling apps doesn't match LightCraft (wxs header comment about .pcraft/PSD associations that the MSI doesn't create; Info.plist UTIs for .pcraft/.psb/.qoi; metainfo psd/layers keywords; nix comment "no git sources" vs craft-ui git dep). Cosmetic; does less than stated, not more.
- F6 (Info): xtask `ci` reads /proc/meminfo / runs powershell Get-CimInstance / vm_stat / sysctl to size build jobs; bench runs `uptime`. Dev tooling, not sandbox detection.

## Conclusion
Clean. All assigned files reviewed in full (tier 3 files proved capability-free then skimmed). No malware behaviour: no hidden network endpoints, telemetry, exfiltration, persistence, obfuscated blobs, time bombs, anti-analysis, or build-time network/exec. The only unsafe is the documented single macOS FFI call in crates/sysmem. One new medium-severity design issue (F1) extends the known control-server exposure.
