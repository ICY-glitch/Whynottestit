# S14 — dependency supply chain (deeper)

Agent S14. Scope: build scripts, proc-macro crates, RustSec matches, yanked versions, craft-ui git dep.

## Progress log

- `cargo fetch --locked` completed (exit 0). Cargo.lock: 665 packages, 1 git source (storytold/craft-libs @ 5c874d51eeb3de70e735c8ce87a59a7dedd47675).

## Step 2 — inventory
- 92 third-party crates with build scripts, 44 proc-macro crates (list computed from Cargo.lock + fetched Cargo.toml; scripts in scratchpad/s14/scan.py).
- Build-script capability grep output saved in scratchpad/s14/bsgrep.txt.

## Step 3 — build scripts reviewed (batch 1: short, read fully)
All read fully, benign (cfg emission / link-search / pkg-config / autocfg / version_check / cfg_aliases only):
ahash 0.8.12, ash 0.38.0 (VULKAN_SDK link-search, feature `linked` only), async-io 2.6.0, crossbeam-deque 0.8.8,
generic-array 0.14.7, getrandom 0.4.3, glutin 0.32.3, jni 0.22.4, jni-macros 0.22.4 (rustc_version), khronos-egl 6.0.0,
libm 0.2.16 (main; configure.rs pending), matrixmultiply 0.3.11, memoffset 0.9.1, muda 0.21.0, naga 30.0.1,
num-bigint-dig 0.8.6, num-traits 0.2.19, orbclient 0.3.55, parking_lot_core 0.9.12, rayon-core 1.13.0, rfd 0.17.2,
rustls 0.23.45, slotmap 1.1.1, smithay-client-toolkit 0.19.2/0.20.0, wayland-backend 0.3.17 (cc on bundled log_shim.c),
wayland-client 0.31.15, wayland-sys 0.31.11, wgpu-core 30.0.1, wgpu-hal 30.0.1, winit 0.30.13.
windows_* (16 crates, 0.52.6 + 0.53.1): all 16 build.rs byte-identical (md5 33187278...), only emit
`rustc-link-search=native=$CARGO_MANIFEST_DIR/lib` (standard windows-rs import-lib crates).
Batch 2 (read fully, benign): android-activity 0.6.1 (cc on bundled android-games-sdk, only with `game-activity` feature),
crc32fast 1.5.2 / object 0.37.3 / paste 1.0.15 / quote 1.0.47 / zmij 1.0.23 (run `$RUSTC --version` only),
crossbeam-epoch 0.9.21, crossbeam-utils 0.8.23, getrandom 0.3.4 (`$RUSTC -vV` via RUSTC_WRAPPER, windows only),
glutin-winit 0.5.0, glutin_egl_sys 0.7.1 / glutin_glx_sys 0.6.1 / glutin_wgl_sys 0.6.1 (gl_generator -> OUT_DIR),
jni-sys 0.3.1 (JAVA_HOME only under `_JNI_SYS_TEST` compile-time env), khronos_api 3.1.0 (include_bytes! of its own
api_webgl/extensions/*.xml), libfuzzer-sys 0.4.13 (cc on own libfuzzer/*.cpp only with `link_libfuzzer`), objc2 0.6.4,
serde_json 1.0.151, wasm-bindgen 0.2.129, wasm-bindgen-shared 0.2.129 (runs `git rev-parse HEAD` in crate dir to stamp
WBG_VERSION; hashes own src/lib.rs + src/tys.rs — harmless), wgpu 30.0.1, wit-bindgen 0.57.1 (wasm only: copies its
bundled prebuilt src/rt/libwit_bindgen_cabi.a into OUT_DIR — upstream-standard prebuilt object, noted as Info),
x11-dl 2.21.0 (pkg-config libdir -> OUT_DIR/config.rs), libm configure.rs (reads CARGO_* env; env::vars() filtered to
CARGO_FEATURE_* only).
Batch 3 (dtolnay-style rustc probe scripts, read fully, benign): anyhow 1.0.104, proc-macro2 1.0.107, thiserror 1.0.69,
thiserror 2.0.21, serde 1.0.229, serde_core 1.0.229 — spawn only $RUSTC (optionally via RUSTC_WRAPPER /
RUSTC_WORKSPACE_WRAPPER) to compile a bundled probe file into OUT_DIR/probe and then remove_dir_all(OUT_DIR/probe);
write OUT_DIR/private.rs. No other paths, no net.
Batch 4 (read fully, benign): rustix 0.38.44 / 1.1.5 (rustc feature probes via stdin, output to OUT_DIR probe file or
stdout-null, removes only OUT_DIR/rustix_test_can_compile), rustversion 1.0.23 (build/build.rs + build/rustc.rs: rustc
--version -> OUT_DIR/version.expr), libc 0.2.189 (rustc --version; `freebsd-version` only when LIBC_CI set; runs
`emcc -dumpversion` from PATH unconditionally — upstream behaviour, harmless probe; reads WIND_RELEASE_ID).
Batch 5 (read fully / non-comment lines, benign): objc-sys 0.3.5 (cfg + cc on extern/exception.m only w/ feature),
curve25519-dalek 4.1.3 (rustc_version), unicode-general-category 1.1.0 (compiles own src/tables.rs into OUT_DIR/category.rs),
crunchy 0.2.4 (writes unroll! macro text to OUT_DIR/lib.rs), zerocopy 0.8.59 (reads own Cargo.toml + rustc --version),
pulp 0.21.5 / 0.22.3 (generates x86 global_asm! of masked vmovups/vinsert/vextract load/store stubs into
OUT_DIR/x86_64_asm.rs — SIMD helper, only vector moves + `jmp rcx`; no syscalls), portable-atomic 1.15.0 (+ version.rs,
src/gen/build.rs: target lists + `rustc -vV`), portable-atomic-util 0.2.8.
Batch 6 (benign): rav1e 0.8.1 / av-scenechange 0.14.1 — `asm` feature NOT active (nasm-rs not in Cargo.lock), so the
build only runs `built::write_built_file()` (rav1e) and emits rustc-env; built 0.8.1 (no git/chrono features in lock)
reads env::vars_os() into a map but writes only CARGO_PKG_*, TARGET, HOST, PROFILE, RUSTC, RUSTDOC, OPT_LEVEL, NUM_JOBS,
DEBUG, features, CI platform name, dependency list from Cargo.lock, `$RUSTC -V` into OUT_DIR/built.rs (compiled into
rav1e; the RUSTC path may embed the builder's toolchain path — standard build metadata, no exfil).
ring 0.17.14 — packaged build (no .git) uses bundled `pregenerated/` asm + objects (104 files incl. prebuilt *-nasm.o
for Windows; upstream-standard), cc compiles bundled BoringSSL-derived C; perl/nasm only run in `.git` dev builds or
RING_PREGENERATE_ASM=1; writes only OUT_DIR/ring_core_generated. Benign.

## Integrity checks
- sha256 of all 636 downloaded .crate files == Cargo.lock checksums (0 mismatches).
- Every file in ~/.cargo/registry/src/<crate> byte-identical to its .crate tarball member; no extra files besides .cargo-ok.
  So the sources reviewed are exactly what crates.io serves for the locked versions.
- Build-script helper libs (run inside build scripts): cc 1.5.1, pkg-config 0.3.34, autocfg 1.5.1, version_check 0.9.5,
  cfg_aliases 0.2.2, rustc_version 0.4.1, semver 1.0.28, gl_generator 0.14.0, walkdir 2.5.0, find-msvc-tools 0.1.14,
  shlex 2.0.1, jobserver 0.1.35, xml-rs 0.8.29: grep for net (std::net/TcpStream/http clients) = none; Command::new only
  compilers/rustc/pkg-config/vswhere (shlex "bash"/"ssh" and jobserver "make" hits are in docs only). Benign.

## Step 4 — proc-macro crates (44)
Capability grep over all non-test .rs (scratchpad/s14/pmgrep.py, pmgrep.txt). Results:
- No proc-macro crate uses std::net / sockets / HTTP clients. No process spawning outside #[cfg(test)]
  (wayland-scanner rustfmt in cfg(test) fn; windows-implement rustfmt in src/tests.rs). zerocopy-derive fs::write only
  in #[cfg(test)] mod output_tests.
- File reads by design: document-features (reads $CARGO_MANIFEST_DIR/Cargo.toml[.orig] to render feature docs),
  wayland-scanner (reads protocol XML given by macro caller, relative to CARGO_MANIFEST_DIR), zbus-lockstep-macros 0.5.2
  (less-known publisher; read lib.rs fully: reads *.xml in the dir named by the macro arg via zbus_lockstep::resolve_xml_path,
  emits a #[cfg(test)] signature test; no writes/net) — benign.
- env reads: paste 1.0.15 and pastey 0.1.1 (fork of paste, as1100k) `[<env!VAR>]` segment -> std::env::var(name given in
  macro input) — documented paste feature, input-driven, no exfil. rustversion reads CARGO_PKG_NAME.
- Less-known publishers skimmed with zero capability hits: arg_enum_proc_macro, interpolate_name, noop_proc_macro,
  simd_helpers (lu-zero/rav1e), async-recursion, dyn-stack-macros, enum-as-inner, enumflags2_derive, equator-macro
  0.1.9/0.4.2, foreign-types-macros, num_enum_derive, phf_macros, profiling-procmacros, jni-sys-macros — pure token
  manipulation (syn/quote).
- Well-known publishers (dtolnay, serde-rs, tokio-rs, rust-lang, RustCrypto, microsoft, unicode-org, google, z-galaxy,
  smithay, taiki-e, Lokathor, jni-rs, wasm-bindgen, dalek, rust-num): grep clean apart from the above.
Verdict proc-macros: clean.

## Step 5 — RustSec (advisory-db HEAD 57dcc0f2, 2026-10-12; matcher scratchpad/s14/scripts/rustsec.py, semver caret/tilde/range, patched+unaffected honoured, withdrawn skipped)
9 matches:
| crate@ver | ID | type | path in LightCraft | affected? |
|---|---|---|---|---|
| paste 1.0.15 | RUSTSEC-2024-0436 | unmaintained | proc-macro via faer-core/gemm (denoise), candle (segment) | no vuln; informational |
| ttf-parser 0.25.1 | RUSTSEC-2026-0192 | unmaintained | ab_glyph -> engine (watermark text) | informational |
| quick-xml 0.38.4 | RUSTSEC-2026-0194 | vuln (DoS, quadratic dup-attr check) | lightcraft-meta XMP/GPX | PARTLY: xmp.rs:231 and gpx.rs:122 use `.with_checks(false)` (safe) but xmp_merge.rs:117 `for a in e.attributes()` uses default checks on the existing sidecar when saving -> crafted sidecar with a huge single start tag can pin a CPU. Element count capped (200k) but not attributes. Low. |
| quick-xml 0.38.4 | RUSTSEC-2026-0195 | vuln (NsReader alloc DoS) | same | NO: only quick_xml::Reader used, no NsReader. |
| rsa 0.9.10 | RUSTSEC-2023-0071 | vuln (Marvin timing, no fix) | rustls-rustcrypto 0.0.2-alpha -> lightcraft-fetch TLS | NO in practice: client uses `.with_no_client_auth()`, so only RSA public-key verification of server signatures; Marvin needs private-key ops. |
| rustls-webpki 0.102.8 | RUSTSEC-2026-0049/0098/0099/0104 | vuln (name constraints, CRL) | rustls-rustcrypto uses it only for `webpki::alg_id` constants | NO: rustls 0.23.45's WebPkiServerVerifier uses rustls-webpki 0.103.15 (patched) for chain validation. |
Note: lightcraft-fetch's TLS crypto provider is `rustls-rustcrypto = "0.0.2-alpha"` (crates/fetch/Cargo.toml), an alpha,
unaudited provider. Model integrity additionally pinned by SHA-256 per doc/description. Low (quality), not malicious.

## Step 6 — yanked check
Sparse index (index.crates.io) fetched for all 561 crate names / 636 locked versions: 0 yanked, 0 missing, and every
index `cksum` equals the Cargo.lock checksum.

## Step 7 — craft-ui git dependency (storytold/craft-libs @ 5c874d51eeb3de70e735c8ce87a59a7dedd47675)
- Root Cargo.toml:13 pins the full 40-hex rev; Cargo.lock source matches. Checkout ~/.cargo/git/checkouts/craft-libs-d830b991a08c6c3d/5c874d5:
  `git rev-parse HEAD` == pinned rev, `git status` clean (only cargo's .cargo-ok), `git fsck` clean; HEAD commit
  "UI.8: shape dock action labels with logical accessibility" (skylar taylor-barrick, 2026-10-10).
- 41 tracked files. No build.rs, no proc-macro; deps egui 0.36 (default-features=false) + optional serde. Workspace
  `unsafe_code = "forbid"`.
- Read all non-test src: lib.rs, icons/mod.rs (string substitution on app-embedded SVG bytes; no file reads),
  theme/mod.rs, buttons/mod.rs, tabs/mod.rs, layout/{mod,geometry,resize,toolbar}.rs, docking/{mod,targets,model}.rs
  fully; docking/surface.rs skimmed fully for API use (egui ctx data/memory/focus only). No fs/net/process/env/unsafe/
  static initialisers/threads. Tests (dev-only, not built into LightCraft) write screenshots to dirs named by
  CRAFT_UI_VISUAL_DIR / CRAFT_UI_DOCK_VISUAL_DIR env vars only. Verdict: clean.
- Its CLAUDE.md/AGENTS.md are ordinary contributor notes (no injection content).

## Other observations
- Repo .cargo/config.toml only defines `xtask` alias; no [patch]/[replace]/[source] overrides. First-party build.rs
  (apps/lightcraft, crates/engine, crates/ui-egui) out of S14 scope (third-party only).
- Unfamiliar names checked: zcheapstr/endi (z-galaxy/zeenix, zvariant deps), polycool (linebender/kurbo), glifo
  (linebender/vello), dify + open + is-wsl + is-docker (dev-only via egui_kittest; is-docker reads /.dockerenv and
  /proc/self/cgroup, is-wsl reads /proc/version — used by `open` to pick a launcher, not shipped in app builds),
  dhat/mintex (opt-in `dhat-heap` feature of lightcraft-cli), winresource (lightcraft build-dep, Windows resources).
- Prebuilt binaries shipped inside deps (upstream-standard): ring pregenerated *-nasm.o (Windows), wit-bindgen
  src/rt/libwit_bindgen_cabi.a (wasm only), windows_* import libraries. Checksums match crates.io.

## Verdict S14: clean (no malicious build scripts / proc-macros / git dep). Findings: 2 Low, 3 Info.
