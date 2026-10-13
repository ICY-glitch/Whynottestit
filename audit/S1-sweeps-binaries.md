# S1: repo-wide sweeps + official release binaries (v0.6.0)

Auditor: S1 agent. Scope: all tracked files (git ls-files, 735 files) for checklist E, F, G, H, L + date constants,
env-gated behaviour, hosts/URLs/IPs, long sleeps, self-copy/self-delete; plus v0.6.0 release binaries.

## Progress log

### Sweep E (low-level / OS calls) — DONE
Pattern: `unsafe|extern "|asm!|global_asm|libc::|windows::|windows_sys|winapi|objc::|msg_send|libloading|dlopen|dlsym|LoadLibrary|GetProcAddress|no_mangle|#[link(`.
- `crates/sysmem/src/lib.rs:28-43`: only production `unsafe` — one `extern "C" malloc_zone_pressure_relief(null, 0)`,
  cfg(macos). Called by `apps/*/src/alloc_release.rs` via engine memory hook. Justified (memory relief, no data access).
- `crates/engine/examples/render_bench.rs:30`, `crates/raster/examples/blur_bench.rs:14`: `libc::clock_gettime` in
  dev examples; `libc` is dev-dependency only (engine/raster Cargo.toml `[dev-dependencies]`). Justified.
- `crates/ui-egui/examples/{grid,thumb_request}_bench.rs`: counting GlobalAlloc wrapper around System in examples. Justified.
- `apps/lightcraft/src/main.rs:368`: `std::os::windows::process::CommandExt::raw_arg` for `explorer /select,"path"`
  (path quoted; see B-sweeps owner; the known cmd /C start issue is ~437). Not low-level.
- `crates/engine/src/lib.rs:15`: `extern crate ... as` rename only.
- No asm!, no_mangle, link attrs, libloading, objc msg_send, windows/winapi crates as direct deps.
- All crate roots carry `#![forbid(unsafe_code)]` except `crates/sysmem` (by design) and `xtask/src/main.rs` (dev tool;
  grep shows zero `unsafe` in xtask/src). nix files mention dlopen of Vulkan/xkbcommon (system libs loaded by
  winit/wgpu deps), justified.

### Sweep F (runs without being asked) — DONE
Pattern: `link_section|#[used]|init_array|ctor|lazy_static|LazyLock|OnceLock|once_cell|set_hook|take_hook|static X: = ` + thread spawns.
- No `#[link_section]`, `#[used]`, `.init_array`, `ctor` crate anywhere. Rust has no static constructors otherwise.
- `static`/`OnceLock` hits (~80): lazily-initialised LUTs, caches, counters, thread pools, mutexes. All lazy, none run at load. Justified.
- `apps/lightcraft-cli/src/main.rs:99-101` `dhat::Alloc` global allocator only under `feature = "dhat-heap"` (heap profiler, off by default).
- Panic hooks: `crates/engine/src/guard.rs:31-41` (appends "secs [thread] panic at file:line: msg" to the given log
  path, `lightcraft-panics.log` in temp), `apps/lightcraft/src/logging.rs:326-335` (chains to app log file),
  `apps/lightcraft-web/src/safety.rs:76-91` (shows error on page). Local only, no network, no extra work. Justified.
- Background threads at/after launch: `lc-gpu-init` (`crates/gpu/src/lib.rs:249`, creates wgpu device),
  `lightcraft-devices` (`crates/engine/src/devices.rs:96`, lists children of mount parents / D:..Z: and checks for a
  `DCIM` folder — camera/card import detection; only `is_dir` / read_dir of the volume root), `lc-availability`
  (offline-original probe of catalog paths), `catalog-snapshot` (journal snapshot write). All match stated purposes.

### Sweep G part 1 (date/time constants, env-gated behaviour) — DONE
- `SystemTime::now` (14 non-test hits): log timestamps, `.corrupt-<secs>` names when setting aside damaged settings
  (`apps/lightcraft/src/main.rs:226`, `crates/engine/src/library.rs:232`), GPU/denoise crash-guard marker contents
  (`crates/gpu/src/backend.rs:169`, `crates/engine/src/denoise.rs:216`), catalog "now" strings, lock-owner `since`.
  None is compared with a constant date. No 10-digit unix-time constants outside tests.
- Date literals outside tests: only `crates/catalog/src/rules.rs:597` and `crates/engine/src/import.rs` wasm-only
  fallback clock "2026-01-01T00:00:00" (wasm has no std clock). Not a trigger.
- Time-based removal checked: `crates/catalog/src/local.rs` `forget_local_plan` (30 days default) removes only
  untouched *catalog records* of Local-browsed folders; `crates/engine/src/cmd/browse.rs:101-123` applies catalog ops
  only ("nothing on disk changes"). Justified, not file destruction.
- Env vars (~80 distinct names): all LIGHTCRAFT_*/LC_* config or test/bench knobs, CARGO_* build vars, HOME/USERPROFILE/
  XDG_DATA_HOME (config dirs), RUST_LOG, WGPU_*. Notables:
  - `USER` (`crates/engine/src/devices.rs:41`): builds /media/$USER, /run/media/$USER for card detection. Justified.
  - `COMPUTERNAME`/`HOSTNAME`/`/etc/hostname` (`crates/catalog/src/lock.rs:72-75`): written into the library lock
    file to say "open in LightCraft on <host>". Local only. Justified (info: hostname is stored in the library folder).
  - `LC_DEMO_BROKEN_GLOBAL_POOL`, `LC_LOCK_CHILD_DIR`, `GTM_FILE`, `LC_SHADER`, `LC_MERGE_DUMP`, `LC_DENOISE_*`,
    `LC_FACE_MODELS`: all inside #[test]/#[ignore] tests (self-spawned child tests). Not in product behaviour.
  - `LIGHTCRAFT_SAM3_TOKEN` (via `token_env` in `crates/fetch/src/lib.rs:348`), `LIGHTCRAFT_SAM3_MIRRORS`,
    `LIGHTCRAFT_SAM3_REMOTE`: known (doc.md). No env var unlocks hidden/unstated behaviour.
- Randomness: no `rand`, `getrandom`, `thread_rng`, `RandomState`, `fastrand`, `OsRng` outside tests/proptest. No
  launch/run counters.

### Sweep: long sleeps — DONE
- No `sleep` > 60 s in product code. `from_secs(60..3600)` hits are timeouts/deadlines (control-server reply 60 s,
  GPU readback 60 s, MCP socket read 90 s, CLI export wait up to 3600 s while an export is running), not sleeps.
  `crates/catalog/src/tests_lock.rs:92` sleeps 120 s in a test child that holds a lock. Justified.

### Sweep: self-copy / self-delete — DONE
- `current_exe` only in: tests (`crates/catalog/src/tests_lock.rs:56`, `crates/engine/src/demo.rs:618` re-run the test
  binary for an isolated test) and `crates/engine/src/library.rs:196` (file stem for the lock-owner label). Never
  followed by copy/remove/rename. No self-deletion or self-copy anywhere. Scripts' `$0` only print their own help.

### Sweep H (obfuscation) — DONE
- `include_bytes!`/`include_str!`: app icons (PNG), Inter fonts, camera-profile JSONs (`assets/camera-profiles/*.json`,
  colour matrices + hue/sat tables with sample counts), `assets/camera-tone/sony-dro-off.json` (tone knots + list of
  CC0 raw.pixls.us source files with sha256), WGSL shaders, locale JSONs, whats-new.md, craft-fonts via build.rs. All readable/expected.
- Binary files: every tracked PNG/JPG/TTF parsed: PNGs contain only IHDR/IDAT/IEND, 0 bytes after IEND; JPEGs 0 bytes
  after EOI; TTFs 0 bytes past the last table. icns/ico/tiff/.DS_Store identified by `file` as what they claim.
- Long hex (>=48): all SHA-256 pins (model downloads, corpus files, known-answer tests). No base64 blobs (only hits are
  git commit hashes inside URLs in ATTRIBUTION.md).
- XOR / "decrypt": `crates/raw/src/vendor/sr2.rs` (Sony SR2SubIFD keystream, SEED[127] u32, recurrence k[n]=k[n-63]^k[n-127])
  and `crates/raw/src/vendor/arw.rs:86` (DECIPHER = inverse of p^3 mod 249) — decode Sony raw maker-note metadata in
  memory; only callers are ARW parsing (`arw.rs:456,640`). Well documented, matches purpose. Other `^=` are test
  corruption (`bad[at] ^= 0xff`), hash-key mixing in `crates/engine/src/media.rs:702-751`, xorshift PRNG in merge/ransac.
- Large byte arrays: JPEG standard Huffman tables (`crates/codecs/src/jpeg_par.rs:32,42`), CFA patterns. Base64 code:
  `crates/develop/src/segmask.rs` (mask storage) and `crates/mcp/src/base64.rs` (MCP image content). Justified.
- Scripts/CI/web: no eval/IEX/-EncodedCommand/atob/new Function/pickle/marshal. `curl https://sh.rustup.rs | sh` in
  CI only (known: unpinned CI tool downloads). `base64 --decode` / `FromBase64String` only for the signing cert from CI
  secrets (`packaging/macos/import-cert.sh:26`, `packaging/windows/sign.ps1:66`). Justified.
- Persistence quick check (I): MSI writes only HKLM App Paths + Software\LightCraft\InstallFolder and a Start-menu
  shortcut; "Launch on Finish" via WixUnelevatedShellExec (UI only). deb/rpm postinst only refreshes desktop/MIME/icon
  caches. No autostart/LaunchAgents/cron/systemd/Run keys/chmod/setuid in Rust code.

### Sweep L (sandbox / VM / debugger detection) — DONE
- No hits for VMware/VBox/VirtualBox/KVMKVMKVM/Microsoft Hv/hypervisor/Xen/.dockerenv/cgroup/cpuinfo/DMI/Sandboxie/
  SbieDll/wine_get_version/cuckoo/IsDebuggerPresent/CheckRemoteDebuggerPresent/ptrace/TracerPid/
  NtQueryInformationProcess/cpuid/GetTickCount in any tracked file (besides audit docs).
- QEMU only in `.github/workflows/release.yml:282-337` (riscv64 cross smoke test under qemu-user). Justified.
- System facts read: `/proc/meminfo` MemTotal or `sysctl -n hw.memsize|hw.physmem` (`crates/engine/src/memory.rs:37-57`)
  → memory budget = RAM/4 clamped 256 MiB..1.5 GiB; never gates any other behaviour. `uptime` load average only in
  `xtask/src/bench.rs:53` (dev bench record). Help ▸ System Info (`crates/ui-egui/src/menus.rs:1067+`) shows OS/arch/
  CPU threads/GPU name locally in a dialog. Justified.

### Sweep: hosts / URLs / IPs in code — DONE
- All URL literals in .rs (non-test) classified: XMP/RDF namespace URIs (ns.adobe.com, w3.org, purl.org, ns.apple.com,
  ns.google.com, iptc.org, cipa.jp, metadataworkinggroup.com, topografix.com, own `ns.lightcraft.app` in
  `crates/meta/src/xmp.rs:18` — identifiers inside XMP, never contacted); pinned model downloads (github.com/opencv/
  opencv_zoo @25f423d, huggingface.co/fal/AuraFace-v1 @af6d057, github.com/darktable-org/darktable-ai release-5.6.0);
  Help links (github.com/storytold/lightcraft, getartcraft.com, discord.gg/artcraft); openstreetmap "Show on Map"
  (known); raw.pixls.us CC0 corpus in xtask; example.org/.com/.invalid and 127.0.0.1 in tests; 127.0.0.1:8080 xtask web.
  Everything matches doc.md's endpoint table.
- Other hostname-like strings (raw.githubusercontent.com for AppStream screenshot / flatpak tool comment, nuget,
  digicert/microsoft timestamp, azure codesigning, flathub, ubuntu ports, nfpm.goreleaser.com, storyteller.ai maintainer
  email, format-doc sites in comments: lclevy.free.fr, photonstophotos.net, capnfabs.net, libopenraw, exiftool.org,
  sqlite.org, helpx.adobe.com) are in comments, packaging metadata or CI. Nothing unexplained.
- IP-like strings: only spec section numbers (heif `7.3.2.2`, `9.3.4.4`), version `1.0.0.0`, test UNC `192.168.1.5`.
- No piecewise URL assembly: `concat!` only for XMP text (gainmap.rs:522), OUT_DIR includes; no strings from byte
  arrays besides base64 alphabet and IPTC latin-1 decode. URL acceptors require `https://` (`faces/src/catalog.rs:91`,
  `denoise-core/src/manifest.rs:95`, `apps/lightcraft/src/main.rs:449` open-URL guard).
- build.rs files (apps/lightcraft, crates/engine, crates/ui-egui): no Command/network; write only OUT_DIR. (J owned by others.)

## Binaries (v0.6.0)
Downloads into scratchpad/s1-bin/<asset>/; all four archives match SHA256SUMS.txt exactly:
- windows-x64-portable.zip 404846d8…f3e, windows-arm64-portable.zip df45d663…27, linux-x86_64.tar.gz d738f16b…f1,
  lightcraft-cli macos-universal.zip e87014d4…ce. Extracted with python3 -I (zipfile / tarfile filter='data'); no path
  traversal, contents = 2 exes + licences/README/NOTICE (+ icons, .desktop, metainfo, mime in the Linux tarball).

### Windows PE imports (x64 + arm64, lightcraft.exe 314 imports, lightcraft-cli.exe 171)
- DLLs: kernel32, user32, gdi32, advapi32, ntdll, ws2_32, bcrypt(+primitives), combase, ole32, oleaut32, shell32,
  dwmapi, dxgi, opengl32, imm32, uxtheme, setupapi, uiautomationcore, api-ms-win-core-synch. No wininet/winhttp/urlmon/
  crypt32/wtsapi/psapi/dbghelp/netapi.
- ABSENT: SetWindowsHookEx*, CreateRemoteThread, WriteProcessMemory, ReadProcessMemory, VirtualAllocEx, OpenProcess,
  any Reg* (no registry API at all), Cert*/Crypt*, CheckRemoteDebuggerPresent, NtQueryInformationProcess,
  URLDownloadToFile, WinHttp*, InternetOpen*, ShellExecute*, CreateToolhelp32Snapshot, AdjustTokenPrivileges, service/task APIs, BitBlt.
- Explained notables:
  - `IsDebuggerPresent` + `UnhandledExceptionFilter`: x64 only, absent on arm64 → MSVC CRT `__scrt_fastfail`/
    `__report_gsfailure` boilerplate, not anti-debug.
  - `SendInput` (GUI only): winit 0.30.13 `force_window_active` (simulated Alt press so SetForegroundWindow works,
    winit window.rs:1571) and muda 0.21.0 native menu (mod.rs:1170). Library behaviour.
  - `RegisterRawInputDevices`/`GetRawInputData`/`GetAsyncKeyState`/`GetKeyboardState`: winit input. Source never calls
    `listen_device_events`, so winit default `DeviceEvents::WhenFocused` = RIDEV_DEVNOTIFY without RIDEV_INPUTSINK
    (winit raw_input.rs:145): no background key capture.
  - `ImpersonateAnonymousToken`/`RevertToSelf`: clipboard-win 5.4.1 `AnonymousTokenImpersonator` (raw.rs:58), used
    around clipboard calls. Clipboard APIs = egui copy/paste (arboard/clipboard-win).
  - `SetupDi*`: wgpu-hal 30.0.1 dx12 adapter.rs (GPU driver info). `SetWindowDisplayAffinity`: winit content protection API.
  - `CreateProcessW` GUI only (std::process::Command: explorer /select, open file — known cmd /C start issue). The CLI
    has NO CreateProcessW: lightcraft-cli.exe cannot launch processes on Windows.
  - ws2_32: GUI has bind/listen/accept (opt-in loopback control server, known) + connect/getaddrinfo (model downloads).
    CLI imports ws2_32 by ordinal: 1/2/13 (accept/bind/listen) absent; only connect(4), send/recv, getaddrinfo,
    select, sockopts → client only (mcp --connect, downloads).
  - `LoadLibrary*`/`GetProcAddress`/`wglGetProcAddress`/`VirtualProtect`: wgpu (Vulkan/D3D12/DXC/GL loaders), std
    compat shims, CRT. No VirtualAllocEx/WriteProcessMemory to pair with it.
- x64 vs arm64 import sets identical except CRT boilerplate above.

### Linux x86_64 ELF (bin/lightcraft, bin/lightcraft-cli) — stripped PIE, glibc
- NEEDED only libgcc_s, libm, libc, ld-linux (everything else, Vulkan/X11/Wayland/xkbcommon, is dlopen'd by
  wgpu/winit as documented in nix/package.nix). No RPATH/RUNPATH. INIT_ARRAY = 16 bytes (2 entries: compiler
  frame_dummy + Rust std argv init), FINI_ARRAY 8 bytes: standard Rust/glibc, no extra constructors.
- 212 / 200 undefined dynamic symbols: libc/libm/unwind/pthread/sockets/posix_spawn/dlopen. `setuid/setgid/setgroups/
  chroot/fork/execvp/posix_spawnp/pidfd_spawnp` = Rust std `process::Command` machinery (always linked when Command is
  used). No `ptrace`, no `prctl`, no `inotify`, no `XGrabKeyboard`-style imports (X11 is dlopen'd by winit only for
  windowing). GUI-only extras: accept4/bind/listen/getsockname/getpeername/shutdown (opt-in control server),
  kill (Child::kill), getpwuid_r (home dir), setlocale. CLI has no bind/listen/accept → client only.

### macOS lightcraft-cli (universal x86_64 + arm64)
- Load commands: 8 LC_LOAD_DYLIB (libSystem, QuartzCore, CoreGraphics, Metal, Foundation, CoreFoundation, libobjc,
  libiconv), LC_MAIN, LC_CODE_SIGNATURE, standard segments. No LC_LOAD_WEAK_DYLIB/LC_REEXPORT/LC_RPATH, no extra
  dylib injection. Both slices import the same 252-255 symbols (only $INODE64/NOCANCEL suffix differences).
- No Security.framework, IOKit, ApplicationServices/Carbon (no CGEventTap/keylogging), ServiceManagement/SMAppService,
  AppleEvents, no `ptrace`. No bind/listen/accept (client only). `_malloc_zone_pressure_relief` = the one sysmem FFI
  call seen in source. `sysctl/sysctlbyname`: Rust std / wgpu-metal; strings check below. `posix_spawnp`/`fork`/
  `execvp` = std Command (memory.rs runs /usr/sbin/sysctl -n hw.memsize).

### Strings: URLs / hostnames / IPs (all 7 binaries; ASCII + UTF-16LE strings)
- IPs: only `127.0.0.1` (GUI ×4) and `127.0.0.1:7980` (CLI default for `--connect`). No other IP literal.
- URL hosts found, all explained by source or a well-known dependency:
  - Product (match doc.md): github.com/storytold/lightcraft, github.com/opencv/opencv_zoo (pinned models),
    huggingface.co/fal/AuraFace-v1, getartcraft.com, discord.gg/artcraft, www.openstreetmap.org (Show on Map),
    github.com/facebookresearch/sam3 (licence link only; no huggingface.co/facebook/sam3 default mirror in any binary),
    raw.pixls.us (inside the embedded sony-dro-off.json provenance list, never fetched by the app).
    darktable-ai URL is absent from every release binary (denoise download not compiled in), so the shipped set is
    a subset of doc.md's table.
  - XMP namespaces: ns.adobe.com, ns.apple.com, ns.google.com, w3.org, purl.org, iptc.org, cipa.jp,
    metadataworkinggroup.com, topografix.com, ns.lightcraft.app.
  - Embedded font name tables: google.com/get/noto, adobe.com/type (Noto CJK), monotype.com, morisawa.co.jp,
    daltonmaag.com (egui's Ubuntu-Light), rsms.me (Inter), scripts.sil.org (OFL), github fontdasu/googlefonts/notofonts/
    source-foundry (Hack).
  - Library messages / embedded shader comments: docs.rs (rustls error text), registry.khronos.org (ash/Vulkan),
    github.com/gfx-rs/wgpu, emilk/egui, shadertoy.com + blog.demofox.org (egui-wgpu egui.wgsl), ml-explore/mlx,
    ggerganov/ggml & llama.cpp, NVIDIA/cccl, arxiv.org, indico.cern.ch, developer.nvidia.com, en.wikipedia.org
    (candle-metal-kernels 0.9.2 .metal sources, used by crates/segment), dbus.freedesktop.org (zbus).
  - Code-signing PKI inside signatures: microsoft.com/pkiops, oneocsp.microsoft.com, ocsp/crl/certs.apple.com.
  - Brotli static dictionary words produce fake "hosts" (www.years, tos.org, thing.org, the.com, people.in…); font
    glyph names produce `*.cn`/`tonos.top`; candle `mma_op.store_result` → "op.store". Noise, not endpoints.
- Signers: Windows "Learning Machines Inc" (Microsoft ID Verified CS EOC CA 04, known in doc.md); macOS
  "Developer ID Application: Learning Machines LLC (DJ6XS33FX8)" (matches repo's publisher name).
- Env var names in binaries (LIGHTCRAFT_*) = exactly the ones in source; no hidden switches.

### Strings: VM / sandbox / debugger / persistence
- No VMware, VBox, VirtualBox, QEMU, KVMKVMKVM, Microsoft Hv, Xen, docker/.dockerenv, sandbox, SbieDll,
  wine_get_version, cuckoo, joebox, TracerPid, P_TRACED, DMI or cpuinfo strings in any binary. "analysis" hits =
  catalog `Analysis` struct + brotli dictionary. `IsDebuggerPresent` only as the x64 CRT import name.
- System paths: /proc/meminfo, /etc/hostname (lock owner), /proc/self/{exe,cgroup,mountinfo} + /sys/fs/cgroup
  (Rust std available_parallelism), /run/media/ (card detection), /usr/share/color/icc (display ICC), /tmp/.X11-unix,
  /etc/machine-id + /var/lib/dbus/machine-id (zbus 5.19 fdo/peer.rs GetMachineId, Linux GUI only, D-Bus local),
  macOS sysctl names hw.memsize, hw.perflevel*, kern.osproductversion. No anti-analysis use.
- No registry Run keys, schtasks, powershell, LaunchAgents, crontab, systemctl, osascript, certutil, bitsadmin,
  mshta. Windows GUI has `explorer`, `rundll32 url.dll,FileProtocolHandler` (open_url) and `cmd` (known open-file
  issue); `cmd.exe /e:ON /v:OFF /d /c` is Rust std's batch-file argument handling.
- Embedded payload scan: exactly one PE stub/ELF header per file (Mach-O: FAT + 2 slices); other PK\x03\x04 /
  0xfeedfacf hits are immediates inside .text (`cmp eax, …` in zip reader / std backtrace), gzip hits inside
  .gcc_except_table. PE .rsrc = icons + VERSIONINFO only (GUI), none (CLI); no manifest elevation request.
- Linux tarball extras: .desktop (identical to packaging/linux source), empty mime package, SVG icon without
  script/foreignObject/external hrefs.

### Sweep G part 2 (magic file names) — DONE
- `.exists()` checks in product code gate only functional things: crash markers (GPU/denoise set-up), installed
  model files, offline originals, rename/convert collision checks. No magic trigger files.

## Cleanup
- Downloaded archives, extracted binaries and string dumps deleted from scratchpad/s1-bin after analysis.

## Verdict (S1)
CLEAN. Repo-wide sweeps E/F/G/H/L, env-gated behaviour, hosts/URLs/IPs, long sleeps and self-copy/self-delete found
nothing malicious or suspicious; every hit is explained by the stated purpose. The four v0.6.0 release archives match
SHA256SUMS.txt; their 7 executables import nothing for keylogging, injection, persistence, registry, certificate
store or anti-debug (IsDebuggerPresent is MSVC CRT boilerplate on x64 only), contain no VM/sandbox checks, no
embedded payloads, and only loopback IPs. Every hostname in the binaries is explained by source, embedded fonts,
library messages/shader comments, XMP namespaces or code-signing PKI. No new findings beyond those in doc.md
(cmd /C start remains present in the Windows binary, as known).
Info only: the library lock file stores the computer's hostname (crates/catalog/src/lock.rs:72-75) — local, for the
"open on <host>" message.
