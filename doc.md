# Security audit: malware, prompt injection and network endpoints

**Repo:** `icy-glitch/whynottestit` (branch `claude/repo-security-audit-5k5448`)
**Started:** 2026-10-11
**Scope:** all 733 tracked files (about 29 MB), plus git history and git config.
**Approach:** the repo's own descriptions (README, CLAUDE.md, AGENTS.md, docs) are treated as untrusted claims, not
as facts. Every phase is checked against the actual code and files.

Phases run one at a time, in this order. Findings are written into the log below as each phase finishes, so nothing
depends on memory.

## Plan / timeline

| # | Phase | What gets checked | Why this order |
|---|-------|-------------------|----------------|
| 0 | Inventory | File list, sizes, file types, unexpected binaries, hidden files, symlinks, git hooks, `.git/config`, submodules | Cheap. Shows where to look first |
| 1 | Prompt injection in agent-facing text | `CLAUDE.md`, `AGENTS.md`, `README.md`, `ROADMAP.md`, `docs/**`, `.claude/`, other files an AI agent would read. Hidden Unicode (zero-width, bidi/Trojan Source, tag chars), HTML comments, instructions aimed at AI agents (exfiltrate, push, disable checks, fetch URLs, run scripts) | These files are what an agent like me reads first, so they are the most likely injection vector |
| 2 | Build-time code execution | `build.rs` files, `.cargo/config.toml`, `Cargo.toml` (build scripts, patch/replace sections, git dependencies, custom registries), `xtask/**`, `flake.nix`, `nix/package.nix`, `rustfmt.toml`, `clippy.toml` | Runs automatically on `cargo build` / `cargo xtask`, before anyone reviews anything |
| 3 | CI/CD and packaging scripts | `.github/workflows/*`, `packaging/**` (`.sh`, `.ps1`, `.py`, `postinst.sh`, `.htaccess`, `_headers`, plist/entitlements), `tools/**` (install scripts, Python, requirements files) | Shell and Python with network and system access: the usual malware hiding spots |
| 4 | Dependency supply chain | `Cargo.lock`: sources (crates.io vs git vs other registries), checksums, typosquat-looking crate names; Python `requirements*.txt` | A malicious dependency doesn't need any code in this repo |
| 5 | Network endpoints in source | Grep all code for URLs, hostnames, IPs, sockets (`TcpStream`, `TcpListener`, `UdpSocket`), HTTP clients (`ureq`, `reqwest`, `hyper`, `fetch`, `XMLHttpRequest`, `WebSocket`), DNS. Focus crates: `fetch`, `mcp`, `apps/lightcraft/control_server.rs`, `apps/lightcraft-web`, model downloaders (`denoise`, `segment`, `faces`) | Answers "which domains get data sent to or received from" |
| 6 | Dangerous behaviour in Rust source | Process spawning (`Command::new`), `unsafe`, file writes outside the app's own folders, env-var or credential reads, `include_bytes!` of opaque blobs, base64/hex/obfuscated strings, `eval`-like code, telemetry, persistence (autostart, launch agents, registry) | Malware inside the product itself |
| 7 | Web / JS / WASM side | `apps/lightcraft-web/index.html`, `worker.js`, packaging/web headers, any inline scripts or CDN loads | Browser code can beacon out or load remote scripts |
| 8 | Binary and data assets | `png`/`jpg`/`tiff`/`ttf`/`ico`/`icns`/`.DS_Store`/`json`/`tsv`/`jsonl`: magic bytes match extension, no appended payloads, no scripts inside SVG, no oversized blobs | Binaries can carry hidden payloads |
| 9 | Git history | Commit authors, deleted files that were suspicious, secrets/keys ever committed, large removed blobs, odd recent commits | Something removed from HEAD can still be in history |
| 10 | Report | Verdict, findings ranked by severity, full list of domains/endpoints with direction (send / receive / local) and the code that uses each | Final deliverable |

Severity scale: **Critical** (active malware / exfiltration), **High** (injection that tries to make an agent do
something harmful, or remote code execution path), **Medium** (risky but plausibly legitimate), **Low / Info** (worth
knowing, not harmful).

## Findings log

### Phase 0: inventory (done)

- 733 tracked files: 538 in `crates/`, the rest in docs, packaging, apps, assets, xtask, tools.
- **No git hooks** (only `.sample` files), no submodules, no `.gitattributes`, no symlinks.
- `.git/config` is clean: one remote (`github.com/ICY-glitch/Whynottestit`), no `core.fsmonitor`, `core.hooksPath`,
  `url.insteadOf` or credential helpers.
- 9 files have the executable bit, all shell scripts under `packaging/` and `tools/` (reviewed in phase 3).
- **No compiled executables** (ELF, PE, Mach-O) and no archives (zip, gzip, tar) anywhere in the tree.
- Binary files are images (png, jpg, tiff, icns, ico), three Inter fonts and one macOS `.DS_Store`. All have the
  correct magic bytes for their extension. Content check is in phase 8.
- Two Rust files (`crates/raw/src/vendor/cr3.rs:446`, `pef.rs:448`) are detected as "data" by `file`. Cause: raw NUL
  bytes inside byte-string literals in test fixtures (a TIFF header `II*\0…`, a Pentax maker-note magic `AOC\0MM`).
  **Benign**, just unusual style.
- To look at later: `assets/app-icon/lightcraft.svg` is 768 KB, large for an icon (phase 8).

**Result: nothing suspicious.**

### Phase 1: prompt injection in agent-facing text (done)

- **Hidden Unicode scan** (zero-width, bidi overrides / Trojan Source, Unicode tag characters, word joiners, BOMs) over
  every text file. Note: the first attempt silently matched nothing because the shell had no UTF-8 locale. The scan
  was checked against planted test samples and re-run under `C.UTF-8`.
  - Only one hit: a U+FEFF BOM in `crates/engine/src/tests_xmp.rs:793`, inside `<?xpacket begin="…"?>`. The XMP spec
    requires that BOM there. **Benign.**
  - All text files are valid UTF-8.
- **`CLAUDE.md`** read in full. It contains coding rules for agents (never-panic rules, clean-room rules, asset
  licensing, how to run tests). Nothing tells an agent to exfiltrate data, fetch and run remote code, disable
  safeguards, hide actions from the user, or push anywhere unexpected. Several rules are *protective* (for example "never send
  OS-level synthetic input", "never weaken the assets check").
- **`AGENTS.md`** is the same as CLAUDE.md, minus two paragraphs, plus a "Contributor credits" section. That section asks an
  agent to offer its human an opt-in credit entry in `contributors/people.toml`, asking only once, and forbids
  touching anyone else's entry. **Low / info:** it suggests running `python3 ../../craftrules/scripts/contributors.py`,
  a script *outside this repo* that can't be audited here. Don't run it unless you trust that sibling repo.
- No `.claude/`, `.cursor/`, Copilot, MCP or editor config files are committed (`.claude/` is in `.gitignore`).
- **Phrase search** over all md/txt/toml/json/yml/html/xml files for classic injection wording ("ignore previous
  instructions", "system prompt", "you are now", "secretly", "without telling", `[INST]`, `<|im_start|>`, etc.): **0
  hits.**
- **HTML comments in Markdown** (a common place to hide instructions): only 3, all tooling markers in
  `docs/parity-checklist.md` (`<!-- parity:summary -->` and a column-order note).
- **Download-and-run / credential wording in docs:** only normal install docs (MSI flags, Hugging Face token
  for optional SAM 3 weights, release signing secrets in GitHub Actions). One claim to verify in phase 5:
  `docs/ai-masks.md` says `LIGHTCRAFT_SAM3_TOKEN` is "sent only over https, and only to the mirror's own host".
- **AI names in code/data:** only `contributors/contributors.json` (credits list naming Claude/Codex/GPT models as
  contributors) and a GPT-2 tokenizer comment. Locale JSON strings: no agent-directed text.

**Result: no prompt injection found.**

### Phase 2: build-time code execution (done)

- **`.cargo/config.toml`:** only an alias `xtask = "run --package xtask --"`. No custom linker, runner, `rustflags`,
  `[source]` replacement or registry override.
- **`rustfmt.toml` / `clippy.toml`:** formatting and lint settings only.
- **`Cargo.toml` (workspace):** crates.io deps plus **one git dependency**:
  `craft-ui` from `github.com/storytold/craft-libs`, pinned to commit `5c874d5…`. That code isn't in this repo and
  isn't audited here (supply-chain note, phase 4). No `[patch]`, `[replace]`, `links` or custom `build =` keys in any
  manifest.
  - Info: `repository = "https://github.com/storytold/lightcraft"`. This repo looks like a copy or fork of that project.
- **3 build scripts**, all read in full:
  - `apps/lightcraft/build.rs`: on Windows targets only, embeds the icon and version info (`winresource`). Benign.
  - `crates/ui-egui/build.rs`: generates Rust from `locales/*-formats.json` and `contributors/*` into `OUT_DIR`.
    Strings are escaped with `serde_json::to_string` / `{:?}`, so data can't inject code. Reads only repo files.
    Benign.
  - `crates/engine/build.rs`: if `CRAFT_FONTS_DIR` is set, `include_bytes!` the fonts listed in that folder's manifest.
    No network. Benign. (If you point it at an untrusted folder, it embeds whatever is there, but only as font bytes.)
- **`xtask/` (2,700 lines):** process spawns are `cargo`, `git ls-files` / `git rev-parse`, `uptime`, `vm_stat`,
  `sysctl`, `powershell Get-CimInstance` (RAM size, to choose CI job counts), `wasm-bindgen`, `wasm-opt` and `curl`.
  - `curl` runs only on `cargo xtask corpus --download`. All 103 URLs are `https://raw.pixls.us/…` (public raw-sample
    database), each pinned to a SHA-256, and a file that doesn't match is deleted.
  - `cargo xtask web --serve` runs a static file server bound to **127.0.0.1 only**, GET/HEAD only, rejects `..`.
  - File writes stay inside the repo (`target/`, `docs/parity-checklist.md` summary, `Cargo.toml` version bump, an
    `.ico` output path).
- **`flake.nix` / `flake.lock` / `nix/package.nix`:** inputs `NixOS/nixpkgs` and `storytold/craft-fonts` (pinned
  rev + narHash). `postInstall` only copies desktop files, icons and licences, validates them, and wraps binaries with
  `LD_LIBRARY_PATH`. Benign.

**Result: nothing malicious. Nothing runs network code during a normal `cargo build`.**

### Phase 3: CI/CD, packaging and tools scripts (done)

- **GitHub workflows** (`freebsd`, `packaging-lint`, `release`, `windows-arm64`):
  - Every third-party action is **pinned to a full commit SHA** (checkout, upload/download-artifact,
    dtolnay/rust-toolchain, Swatinem/rust-cache, vmactions/freebsd-vm). Good practice.
  - Default `permissions: contents: read`. Only the final "Draft GitHub Release" job has `contents: write`, and it
    only creates or updates a **draft** release with `gh`.
  - No `pull_request_target`, no `workflow_run`, and no untrusted `github.event.*` text spliced into shell
    (script injection).
  - Signing secrets (Apple, Windows, Azure) are passed only to the signing steps, inside a `release` environment.
  - Second checkout: `storytold/craft-fonts` pinned to commit `8dcdacd…` (fonts only).
  - Remote installers: `sh.rustup.rs | sh` (official Rust installer), `actionlint` from `github.com/rhysd/…` and
    `nfpm` from `github.com/goreleaser/…` release pages, Ubuntu ports mirror. **Low:** `actionlint` and `nfpm` are
    fetched without a checksum check. That's common, but pinning a SHA-256 would be safer.
- **`packaging/linux/postinst.sh`** (runs as root on deb/rpm install and remove): only refreshes the desktop, MIME and
  icon caches. Benign.
- **Windows installer (`lightcraft.wxs`):** registry writes only to `HKLM\Software\LightCraft\InstallFolder` and the
  standard `App Paths\lightcraft.exe`. A Start-menu shortcut. One custom action: the optional "Launch LightCraft"
  checkbox on the Finish page, run **unelevated** as the user. No services, Run keys, scheduled tasks or firewall
  rules.
- **macOS:** `entitlements.plist` is **empty** (no JIT, no unsigned code, no camera/mic/network exceptions).
  `Info.plist` holds only standard bundle and file-type keys. `import-cert.sh` decodes the base64 signing cert from CI
  secrets into a temporary keychain (standard practice).
- **Flatpak manifests:** wayland/x11, ipc, GPU, `xdg-pictures`. **No `--share=network` by default** (the manifest says
  network is opt-in with `flatpak override`). That fits an app with no telemetry.
- **`packaging/windows/sign.ps1`:** timestamp servers `timestamp.digicert.com` / `timestamp.acs.microsoft.com`, and
  downloads the `Microsoft.Trusted.Signing.Client` package from `nuget.org`. Standard code signing.
- **`packaging/linux/package.sh`:** downloads `appimagetool` from `github.com/AppImage/appimagetool` (continuous build,
  no checksum: **Low**, same as above). `flatpak-bundle.sh` uses the Flathub repo.
- **`tools/install-sam3.sh` / `.ps1`** (optional, run by hand): read the Hugging Face token from `HF_TOKEN` or
  `~/.cache/huggingface/token` and send it **only to `https://huggingface.co`**. The host is hardcoded; `--repo` only
  changes the path. Downloads Meta's SAM 3 weights and checks a pinned SHA-256 and size. Matches what the docs claim.
- **Python tools:** `sam3_reference.py` (local torch reference tensors, loads `safetensors`, no pickle),
  `spectral_camera_table.py` (reads a local `rawtoaces-data` clone, runs `git rev-parse`), `docs/raw/coverage.py`
  (CSV → Markdown), `docs/showcase/run.py` (talks to the app's control port on **127.0.0.1:7980**),
  `dmg/generate.py` (builds the DMG background and `.DS_Store`). No network calls, no `eval`/`exec`.
- **`requirements-*.txt`:** `torch`, `torchvision`, `transformers`, `pillow`, `safetensors`, `numpy`, `scipy`,
  `colour-science`. All real, well-known PyPI packages (no typosquats). Only the spectral one pins versions.

**Result: nothing malicious. Only standard build, sign and release tooling.**

### Phase 4: dependency supply chain (done)

- `Cargo.lock`: 665 packages. **636 from crates.io, all with SHA-256 checksums**, 1 git dependency, and the rest
  are the workspace's own path crates. No other registries or mirrors.
- **Typosquat review:** all 636 crate names checked by hand. They're the expected egui / wgpu / winit / zbus / image /
  rustls / candle ecosystem. Names I didn't recognise (`zcheapstr`, `glifo`, `polycool`, `naga-types`,
  `wgpu-naga-bridge`, `pxfm`, `zmij`, `endi`, `dify`, `coe-rs`, `mintex`) were traced in the lockfile. Each is a
  *transitive* dependency of a mainstream crate (zvariant, vello_cpu, kurbo, wgpu, moxcms, serde_json, egui_kittest,
  faer, dhat). Not typosquats.
- **`heic-rs` 0.1.1** (direct dependency of `crates/heif`): **Medium supply-chain risk (by age, not content)**. It's
  one month old (first published 2026-09-12), by a single author (`github.com/tbraun96/heic-rs`), with about
  20 k downloads. I downloaded the published `.crate`: its SHA-256 matches `Cargo.lock`. It has **no build
  script**, `#![forbid(unsafe_code)]`, no network, process, env or file-write calls, and its only dependency is `rayon`.
  It looks clean today. Re-review it whenever the pinned version (`=0.1.1`) is bumped.
- **`rustls-rustcrypto` 0.0.2-alpha** (TLS crypto for the model downloader): official RustCrypto project, alpha but
  legitimate (230 k downloads).
- **Git dependency `craft-ui`** (`github.com/storytold/craft-libs` @ `5c874d5`, a shallow clone of exactly that commit):
  a small egui widget, icon and docking library. Dependencies are `egui` and optional `serde`. No build script, no
  `unsafe`, no network, process, env or file-write calls, no hidden Unicode, no injection text in its
  CLAUDE.md/AGENTS.md/README. Clean.
- Python requirements: see phase 3 (all real packages).

**Result: no malicious or typosquatted dependencies. One young single-author crate (`heic-rs`) to watch.**

### Phase 5: network code in the product (done)

Only **four places** in the product open network connections:

1. **`crates/fetch`**: the only outbound HTTP(S) client (pure-Rust `rustls`, Mozilla `webpki-roots`, normal certificate
   verification, no "accept any cert" bypass). Used only to download optional AI models.
   - Every catalogued download is pinned to a **commit/revision and a SHA-256 + size**. A mismatch is rejected.
   - Downloads start **only from a command** the user triggers (`denoise`, `faces`, `segment.model.download`). SAM 3
     additionally requires `acknowledged: true` (licence consent). Nothing downloads at startup.
   - **Token claim from the docs verified** (`crates/fetch/src/lib.rs:279`): `LIGHTCRAFT_SAM3_TOKEN` is attached only
     when the URL is `https` **and** its host equals the configured mirror's host. It's never forwarded to a redirect
     target. Max redirects enforced, CR/LF header injection rejected, `file:`/`ftp:` rejected.
   - **SAM 3 has no default download location** (`DEFAULT_MIRRORS = []`). It downloads only from mirrors the user sets
     in `LIGHTCRAFT_SAM3_MIRRORS` or a mirrors file.
   - Low: user-configured mirrors may be plain `http://`, and SAM 3's two tokenizer text files have no pinned hash
     (the 3.4 GB weights do). Only an issue if you point it at an untrusted mirror.
2. **`crates/segment/src/remote.rs`** + `lightcraft-sam3-worker` binary: optional SAM 3 worker protocol (for running
   the model on another Mac over an SSH tunnel). **Both client and server refuse any non-loopback address**
   (`is_loopback()` checks at lines 84 and 189). Enabled only by `LIGHTCRAFT_SAM3_REMOTE`.
3. **`apps/lightcraft/src/control_server.rs`**: JSON-lines automation server. **Off by default.** Enabled only with
   `--control PORT` or `LIGHTCRAFT_CONTROL_PORT`. Binds **127.0.0.1** only. Limits: 16 connections, 4 MB per line.
   - **Low / design note:** no authentication. While it's on, *any local program* can drive the app (import, export,
     edit the library). A malicious *web page* can't: the first non-JSON line (an HTTP request line) closes the
     connection, so browser cross-protocol requests are blocked. Use it only on a machine you trust.
4. **`crates/mcp/src/backend.rs`** (`lightcraft-cli mcp --connect`): connects to that control server, default
   `127.0.0.1:7980`, or an address the user passes.

Other:
- **Links opened in the system browser** (`open` / `xdg-open` / Windows shell) **only on a click**, **https only**
  (`apps/lightcraft/src/main.rs:449`): Help/GitHub/feedback (`github.com/storytold/lightcraft`), website
  (`getartcraft.com`), community (`discord.gg/artcraft`), and **"Show on Map"**, which opens
  `openstreetmap.org/?mlat=…&mlon=…` with **that photo's GPS coordinates** (privacy note: only when you click it).
- **No telemetry, analytics, crash reporting, update checks or licence checks** anywhere in the Rust code. No HTTP
  client besides `crates/fetch`.
- All other URLs in the source are **XML namespace identifiers** (never contacted: `ns.adobe.com`, `w3.org`,
  `purl.org`, `ns.google.com`, `ns.apple.com`, `iptc.org`, `cipa.jp`, `metadataworkinggroup.com`,
  `topografix.com`, `ns.lightcraft.app`), **reference links in comments** (exiftool.org, lclevy.free.fr,
  libopenraw, photonstophotos.net, capnfabs.net, sqlite.org, helpx.adobe.com, gnu.org, modelcontextprotocol.io), or
  test placeholders (`example.com/org`, `a.example`, `example.invalid`, `127.0.0.1:0`).

**Result: no hidden outbound connections. All networking is optional, user-triggered, and either pinned and
verified or loopback-only.**

### Phase 6: risky behaviour in the Rust source (done)

- **`unsafe`:** only `crates/sysmem/src/lib.rs` (one documented macOS libSystem call, `malloc_zone_pressure_relief`,
  which returns freed memory pages to the OS), plus counting allocators and `clock_gettime` in **benchmark examples**
  that don't ship. Matches what CLAUDE.md claims.
- **Process spawning in product code:** only "open this file / URL / folder" helpers (`open`, `xdg-open`, `explorer`,
  `rundll32 url.dll,FileProtocolHandler`, `cmd /C start`), the user's chosen external editor ("Edit in…"), and
  `sysctl hw.memsize` / `hw.physmem` to read RAM size. No shells running downloaded or constructed scripts.
- **Bug, Medium (not malware): Windows command injection via a file name.** `apps/lightcraft/src/main.rs:437-446`:
  opening a photo in the *default* app on Windows runs `cmd /C start "" <path>`. Rust's Windows argument quoting only
  adds quotes when an argument contains a space or tab, and `cmd.exe` treats `&`, `|`, `^`, `<`, `>` as shell
  metacharacters. A photo whose full path has no spaces but contains `&` (for example,
  `C:\Users\bob\Pictures\x&calc&.jpg`, delivered in a zip, on a card, or by import) would run `calc` when the user
  picks "open in default app". Suggested fix: use `rundll32 url.dll,FileProtocolHandler <path>` (already used for
  URLs) or `explorer <path>`, or call `ShellExecuteW`, instead of `cmd /C start`. The same pattern done right is
  `explorer_select_arg` (line 350), which strips quotes and uses `raw_arg`.
- **Environment variables:** about 80 read, all the app's own settings (`LIGHTCRAFT_*`), test/bench knobs (`LC_*`,
  `PHOTO_*`, …), or standard locations (`HOME`, `USERPROFILE`, `XDG_DATA_HOME`, `USER`). The only secret is the
  optional `LIGHTCRAFT_SAM3_TOKEN` (see phase 5).
- **No access to sensitive locations:** no references to `~/.ssh`, `.aws`, `.gnupg`, keychains, browser cookie or
  login stores, crypto wallets, `.netrc`, `.git-credentials`, `/etc/passwd`. "Clipboard" appears only in the
  headless test host.
- **No obfuscation:** no long base64 or hex string literals. Inline byte arrays are standard JPEG Huffman tables
  and raw-format constants. No XOR-decode tricks. All 40-odd `include_bytes!` / `include_str!` targets are visible
  repo files (Inter fonts, icons, WGSL shaders, locale JSON, camera profiles, `docs/whats-new.md`, a corpus
  checksum list).
- **No persistence:** no LaunchAgents/Daemons, login items, autostart `.desktop` files, Run keys, cron or systemd units,
  and no shell-rc edits.

**Result: no malicious behaviour. One genuine security bug (Windows `cmd /C start` file-name injection) to fix.**

### Phase 7: web / JS / WASM side (done)

- **`apps/lightcraft-web/index.html`** (49 lines): one inline module script that imports `./lightcraft_web.js`
  and fetches `./lightcraft_web_bg.wasm` and `./lightcraft_zh_hans.otf`, all **same-origin relative paths**. Favicon is
  `data:,`. **No CDNs, no external scripts or fonts, no analytics/tag managers**, no `eval` / `new Function` /
  `document.write`.
- **`worker.js`** (15 lines): loads the same wasm module in a Web Worker for rendering. Nothing else.
- **Rust web code (`apps/lightcraft-web/src`)**: storage is **local to the browser** (Origin Private File System, then
  IndexedDB, then memory). Communication is `postMessage` to its own workers. Reads `location.search` only for
  `?store=` and similar flags.
- **`web-sys` features compiled in (40):** files, OPFS, IndexedDB, canvas, workers, `Url`, `Location`, `Navigator`
  (for `navigator.storage`). **No `Request` / `Response` / `Headers` / `WebSocket` / `EventSource` / `sendBeacon` /
  WebRTC bindings**, so the WASM app *can't* make network requests beyond the initial same-origin loads in
  `index.html`.
- **Deployment headers (`_headers`, `.htaccess`):** MIME types, `nosniff`, `no-cache`, compression. No redirects or
  proxying to other hosts. Low: no Content-Security-Policy is set; adding one (`default-src 'self'`) would harden a
  deployment.

**Result: the web build is fully offline after load. No outbound endpoints.**

### Phase 8: binary and data assets (done)

- **28 PNG/JPEG files:** parsed to their real end (PNG `IEND` chunk, JPEG final `EOI`). **No trailing appended data**
  in any of them.
- **All 35 binary files** (png, jpg, tiff, icns, ico, ttf, `.DS_Store`) searched for embedded signatures of PE / ELF /
  Mach-O executables, ZIP / gzip / RAR / 7z archives, shebangs, `<script>`, `powershell`, `eval(`, and URLs:
  **no hits.**
- **8 SVGs:** no `<script>`, event handlers (`onload`, `onclick`, …), `javascript:`, `<foreignObject>`, `<iframe>`, or
  external `href`s. The 768 KB `lightcraft.svg` is just two very detailed `<path>` elements (a traced illustration).
  No embedded base64 data.
- **`dmg-layout.DS_Store`:** only Finder window-layout data for the DMG (`/Volumes/LightCraft`, `Applications`,
  `background.tiff`, window bounds). **No usernames or personal paths leaked.**
- **Fonts:** three Inter TTFs with normal Inter metadata (`Copyright 2016 The Inter Project Authors`, version 4.001).
- **Data files** (camera profile JSON, TSV tables, showcase JSONL, contributor JSON): ordinary data. The only URLs are
  `github.com` and `raw.pixls.us`.

**Result: no hidden payloads in assets.**

### Phase 9: git history (done)

- **This repo is an unmodified copy of upstream.** Local `HEAD` `0ba118de…` (tree `7657bd03…`) is **exactly** the
  current `main` / `HEAD` of the public `github.com/storytold/lightcraft` (checked with `git ls-remote` and by
  fetching that commit from upstream: same commit hash, same tree hash). Nothing was added, removed or modified in
  this copy, so any injected change would have been visible here. That makes this a review of upstream LightCraft
  as published.
- The clone is shallow. Visible history: about 1,550 commits from 2026-10-04 to 2026-10-10, by roughly 30 human
  contributors (top: Brandon Thomas, Nathaniel Ledford, radialmonster) plus many AI-agent-authored commits
  (`Claude <noreply@anthropic.com>`, Codex). The newest commit is an upstream PR merge (#869).
  - Side note: my `git fetch --depth 1` of that upstream commit temporarily moved the local shallow boundary and hid
    history. It was restored with a standard `git fetch --deepen`. The working tree was never touched.
- **Files deleted in the visible history (17):** Japanese and Source Sans fonts plus their OFL texts (moved out to
  `storytold/craft-fonts`; Source Sans removed under the "no Adobe fonts" rule), an old `segment.rs`, Russian locale
  files (re-added later), and two **accidental `.claude/worktrees/agent-*` gitlinks** (empty submodule pointers an AI
  agent committed by mistake; no content). `scripts/package-japanese-macos.py` was deleted at the shallow boundary,
  so its content isn't available locally. It isn't in the current tree, so it can't run.
- **Secrets scan** (AWS, GitHub, Hugging Face, OpenAI/Anthropic, Slack, Google API keys, private-key blocks, hardcoded
  passwords) over the full visible history and the current tree: **0 hits.**

**Result: no tampering, no secrets, nothing suspicious removed.**

## Network endpoints

Final. The web build (phase 7) adds **no** endpoints: after loading its own files from the hosting site, it makes no
network requests. Every row below is either off by default, triggered by an explicit user action, or local
(loopback). **Nothing contacts any server at startup or in the background.**

**Shipped app (desktop + CLI):**

| Domain / address | Direction | When | Code |
|---|---|---|---|
| `github.com/opencv/opencv_zoo` (raw files @ commit `25f423d`) | **Receive** (download YuNet + SFace face models, SHA-256 pinned) | User starts a face-model download | `crates/faces/src/known.rs:125-130` → `crates/engine/src/face_download.rs` |
| `huggingface.co/fal/AuraFace-v1` (@ rev `af6d057`) | **Receive** (AuraFace model, pinned) | User starts that download | `crates/faces/src/known.rs:133` |
| `github.com/darktable-org/darktable-ai` (release 5.6.0) | **Receive** (raw denoise model, pinned) | User starts the denoise-model download | `crates/denoise-core/src/known.rs:52` → `crates/engine/src/model_download.rs` |
| User-configured SAM 3 mirror(s) (`LIGHTCRAFT_SAM3_MIRRORS` / mirrors file; no default) | **Receive**. **Sends** `LIGHTCRAFT_SAM3_TOKEN` as a bearer token to that same host only, https only | User runs `segment.model.download` with consent | `crates/segment/src/fetch/mod.rs`, `crates/fetch/src/lib.rs` |
| `127.0.0.1:<port>` control server | **Local** listen (send + receive) | Only with `--control` / `LIGHTCRAFT_CONTROL_PORT` | `apps/lightcraft/src/control_server.rs:28` |
| `127.0.0.1:7980` (default) or user address | **Local** client | `lightcraft-cli mcp/run --connect` | `crates/mcp/src/backend.rs:52` |
| Loopback SAM 3 worker (`LIGHTCRAFT_SAM3_REMOTE`) | **Local** client/server; sends the image + clicks/text prompt | Only when configured | `crates/segment/src/remote.rs:84,189` |
| `www.openstreetmap.org` | **Sends** the photo's GPS lat/lon (opened in your browser) | Click "Show on Map" | `crates/ui-egui/src/panels/right.rs:666` |
| `github.com/storytold/lightcraft`, `getartcraft.com`, `discord.gg/artcraft` | Opened in your browser | Click Help / Website / Community / Feedback | `crates/ui-egui/src/links.rs` |

**Developer/CI only (never in the shipped app):**

| Domain | Direction | Where |
|---|---|---|
| `raw.pixls.us` | Receive (CC0 sample raws, SHA-256 pinned) | `cargo xtask corpus --download` (`xtask/src/main.rs`) |
| `127.0.0.1:8080` | Local dev web server | `cargo xtask web --serve` |
| `huggingface.co/facebook/sam3` | Receive; **sends** your HF token | `tools/install-sam3.sh` / `.ps1` (run by hand) |
| `sh.rustup.rs`, `github.com` (actionlint, nfpm, appimagetool releases, craft-fonts), `ports.ubuntu.com`, `dl.flathub.org`, FreeBSD pkg mirrors | Receive (toolchains / tools) | `.github/workflows/*`, `packaging/linux/*` |
| `timestamp.digicert.com`, `timestamp.acs.microsoft.com`, `nuget.org`, Azure Trusted Signing endpoint | Send file hashes for timestamps / download the signing client | `packaging/windows/sign.ps1` (release CI) |
| Apple notarization service (via `xcrun notarytool`) | Sends the app for notarization | `packaging/macos/package.sh` (release CI) |
| `crates.io`, `static.crates.io`, `github.com/storytold/craft-libs` | Receive (dependencies) | `cargo build` |

## Verdict

**No malware, no prompt injection, no data exfiltration, no telemetry.** This repository is a byte-for-byte copy of
the public `storytold/lightcraft` (same commit and tree). Its code does what it says: a pure-Rust photo library and
raw developer. Networking is limited to optional, user-triggered, checksum-pinned AI-model downloads and a few
loopback-only automation channels.

### Findings, most severe first

| # | Severity | Finding | Where | What to do |
|---|----------|---------|-------|------------|
| 1 | **Medium** (bug, not malicious) | Windows command injection through a file name: "open in default app" runs `cmd /C start "" <path>`. A path with `&` and no spaces makes `cmd.exe` run extra commands. | `apps/lightcraft/src/main.rs:437-446` | Use `rundll32 url.dll,FileProtocolHandler`, `explorer`, or `ShellExecuteW` instead of `cmd /C start` |
| 2 | **Medium** (supply chain, by age) | `heic-rs` 0.1.1 is a one-month-old, single-author crate and a direct dependency. Its source looks clean today (no build script, `forbid(unsafe_code)`, no I/O). | `crates/heif/Cargo.toml` | Re-review on every version bump. It's pinned with `=0.1.1`, which is good |
| 3 | Low | The control server has no authentication. While `--control` is on, any local process can drive the app. Browsers are blocked (the first non-JSON line closes the connection). | `apps/lightcraft/src/control_server.rs` | Off by default. Turn it on only on a machine you trust |
| 4 | Low | CI/packaging downloads `actionlint`, `nfpm` and `appimagetool` (continuous build) without verifying a checksum. | `.github/workflows/packaging-lint.yml`, `release.yml`, `packaging/linux/package.sh` | Pin SHA-256s |
| 5 | Low | User-configured SAM 3 mirrors may be `http://`, and its two tokenizer files have no pinned hash (the weights do). | `crates/segment/src/fetch/mod.rs` | Only matters if you configure an untrusted mirror |
| 6 | Low | The web deployment headers set no Content-Security-Policy. | `packaging/web/_headers`, `.htaccess` | Add `default-src 'self'` (plus `wasm-unsafe-eval`) |
| 7 | Info | `AGENTS.md` suggests running `../../craftrules/scripts/contributors.py`, a script outside this repo. Not injection (opt-in credit, ask once), but don't run unaudited sibling-repo scripts. | `AGENTS.md` | None needed |
| 8 | Info | "Show on Map" sends the photo's GPS coordinates to openstreetmap.org (in your browser, only on click). | `crates/ui-egui/src/panels/right.rs:666` | Privacy awareness only |

### What was checked and came back clean

Hidden Unicode / Trojan Source; injection wording in every doc, config and data file; git hooks, git config and
submodules; build scripts; cargo config; xtask; Nix; all four GitHub workflows (pinned SHAs, read-only tokens, no
`pull_request_target`); installers and post-install scripts (deb/rpm, MSI, DMG, Flatpak); macOS entitlements; tools and
Python scripts; all 665 dependencies (checksums, typosquats, plus the git dependency and `heic-rs` read at source);
every socket and HTTP call in the product; token handling; process spawning; `unsafe`; env/credential access;
obfuscated strings; embedded blobs; persistence mechanisms; the web/WASM build; all 35 binaries and 8 SVGs; git
history for tampering, deletions and secrets.

**Limits of this audit:** dependencies were vetted by name, source and checksum. Only `heic-rs` and `craft-libs` were
read at source, not all 636 crates. Commits before the shallow boundary weren't inspected. Nothing was compiled or
run.
