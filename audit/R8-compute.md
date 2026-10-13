# R8-compute audit notes

Scope: 115 files from audit/chunks/R8-compute.txt (codecs, color, geom, gpu incl. WGSL, merge, pipeline, raster, scenes).
Question: does any of this code behave like malware? Method: capability grep across all files (network, process, fs,
env, unsafe/FFI, link_section, include_bytes, time, rand, encodings, VM/debugger markers, clipboard, registry), then
per-file reading. Tier 1 = touches A-L capabilities (read every line); 2 = ordinary logic; 3 = pure computation.

Global grep results (prod files):
- No std::net / sockets / HTTP clients / Command::new / std::process (except process::id() in test code) in any file.
- No extern/FFI/asm!/link_section/#[used]/libloading/clipboard/registry/VM-or-debugger markers.
- Every production crate root has `#![forbid(unsafe_code)]` (codecs, color, geom, gpu, merge, pipeline, raster, scenes).
  Only `unsafe` = raster/examples/blur_bench.rs libc::clock_gettime (dev example, CPU-time read).
- Production fs access only in crates/gpu/src/backend.rs (GPU-init crash marker file, path supplied by caller).
- `http://ns...` strings are XMP namespace URIs (data, not network).
- env vars read: LIGHTCRAFT_GPU, LIGHTCRAFT_GPU_BACKEND, WGPU_BACKEND, WGPU_DX12_COMPILER, LIGHTCRAFT_GPU_LIMITS,
  LIGHTCRAFT_GPU_ADAPTER, LIGHTCRAFT_PROFILE (config knobs); LC_DENOISE_MODEL/LC_SHADER/LC_BLOCKS only in #[cfg(test)].

| path | lines | tier | caps | status | note |
|---|---|---|---|---|---|
| crates/codecs/Cargo.toml | 50 | 2 | - | done | deps image/zune/jpeg/png/tiff/webp/moxcms/jxl-oxide/ravif; heif optional (known). No build.rs in crate. |
| crates/color/Cargo.toml | 16 | 3 | - | done | serde + geom only; no build.rs |
| crates/geom/Cargo.toml | 15 | 3 | - | done | serde only; no build.rs |
| crates/gpu/Cargo.toml | 33 | 2 | - | done | wgpu/pollster/bytemuck native-only; optional denoise; no build.rs |
| crates/merge/Cargo.toml | 21 | 2 | - | done | internal crates + rayon/serde/thiserror; no build.rs |
| crates/pipeline/Cargo.toml | 24 | 2 | - | done | internal crates + rayon/serde; no build.rs |
| crates/raster/Cargo.toml | 23 | 2 | E(dev) | done | libc is a dev-dependency only (bench example) |
| crates/scenes/Cargo.toml | 18 | 2 | - | done | color/raster/serde/rayon; no build.rs |
| crates/gpu/src/backend.rs | 283 | 1 | B?no C D(env) | done | env knobs choose wgpu backends; DX12 compiler forced to FXC (avoids loading stray dxcompiler.dll from PATH - safety positive). Crash-sentinel marker: write/remove file at caller-given path (apps/lightcraft main.rs: config_dir()/gpu-init.marker); content = backends + unix time. Justified. Test uses temp dir only. |
| crates/gpu/src/lib.rs | 463 | 1 | D(env) F(warm_up thread on request) | done | GPU on/off state, lazy device creation (thread lc-gpu-init only when warm_up() called by app), catch_unwind around render. adapter_name() local string for perf display only. Clean. |
| crates/gpu/src/ctx.rs | 734 | 1 | D(env LIGHTCRAFT_GPU_LIMITS) | done | wgpu device creation (HighPerformance adapter; skips software/CPU adapter = perf decision, not sandbox evasion: just falls back to CPU render). AdapterInfo used only in log/error strings. WGSL assembled from include_str! of the crate's own .wgsl + numeric constants. Buffer pool, readback. No I/O. Non-test assert! at 505/529 (inside catch_unwind) - not security. |
| crates/gpu/src/exit.rs | 132 | 2 | - | done | shutdown gate (mutex+condvar counter). Instant only for timeout. Clean. |
| crates/gpu/src/nn.rs | 1075 | 1 | D(env LIGHTCRAFT_GPU_ADAPTER) | done | denoise U-Net on own wgpu device; adapter chosen by HighPerformance or name substring; adapter names only into local log/error. Shader = include_str!(nn_conv.wgsl) with numeric {{BM}} template substitution. Test mod writes synthetic ONNX to temp dir and removes it; ignored tests read env-named local files (dev probes). Clean. |
| crates/gpu/src/render.rs | 1164 | 2 | D(env LIGHTCRAFT_PROFILE) | done | GPU render graph: buffers, dispatches, readback, histogram; profiling eprintln to stderr only. No I/O. Clean. |
| crates/gpu/src/params.rs | 248 | 3 | - | done | packs FinishParams into u32 words; pure. |
| crates/gpu/src/wgsl/common.wgsl | 67 | 3 | - | done | math helpers (oklab, luminance). WGSL has no I/O capability. |
| crates/gpu/src/wgsl/blur.wgsl | 80 | 3 | - | done | box blur passes on bound buffers |
| crates/gpu/src/wgsl/finish.wgsl | 517 | 3 | - | done | per-pixel tone/colour/grain (hash = film grain noise); entry `main` only |
| crates/gpu/src/wgsl/geom.wgsl | 197 | 3 | - | done | orient/affine/warp resampling; entries match ctx.rs |
| crates/gpu/src/wgsl/map.wgsl | 228 | 3 | - | done | element-wise kernels; 14 entries == ctx.rs MODULES list |
| crates/gpu/src/wgsl/mask.wgsl | 171 | 3 | - | done | mask shape/combine/finalize |
| crates/gpu/src/wgsl/nn_conv.wgsl | 165 | 3 | - | done | tiled conv GEMM for denoise U-Net; {{BM}} etc host-substituted integers |
| crates/gpu/src/wgsl/nn_pool.wgsl | 28 | 3 | - | done | 2x2 maxpool |
| crates/gpu/src/wgsl/resize.wgsl | 51 | 3 | - | done | separable resample |
| crates/gpu/tests/equivalence.rs | 814 | test | - | done | CPU vs GPU render comparison on procedural scenes; no fs/net/process |
| crates/gpu/tests/fallback.rs | 103 | test | - | done | fault injection; no fs/net/process |
| crates/gpu/tests/memory.rs | 41 | test | - | done | buffer pool accounting |
| crates/gpu/tests/shutdown.rs | 58 | test | - | done | shutdown gate |
| crates/gpu/tests/shutdown_first.rs | 25 | test | - | done | shutdown before init |
| crates/codecs/src/lib.rs | 321 | 2 | - | done | decode/header/thumbnail dispatch on in-memory bytes; catch_unwind guards; forbid(unsafe). |
| crates/codecs/src/jpeg_par.rs | 508 | 3 | - | done | parallel baseline JPEG encoder; hex = T.81 Annex K standard Huffman/quant tables; headers SOI/APP0 JFIF/caller APP segments/DQT/SOF/DHT/DRI/SOS/EOI only - no hidden payload. |
| crates/codecs/src/encode.rs | 528 | 2 | - | done | JPEG/PNG/TIFF/WebP/AVIF encoders to Vec<u8>; embeds only caller-provided ICC/EXIF/XMP/ppi (no extra tags, IDs, or fingerprints). XMP_NS http string is namespace id. |
| crates/codecs/src/sniff.rs | 289 | 3 | - | done | magic-number sniffing; bounded IFD walk (seen>16 break) |
| crates/codecs/src/other.rs | 38 | 3 | - | done | GIF/BMP via image crate from in-memory Cursor |
| crates/codecs/src/webp.rs | 56 | 3 | - | done | WebP decode/header from memory; saturating chunk walk |
| crates/codecs/src/jxl.rs | 76 | 3 | - | done | jxl-oxide decode from memory |
| crates/codecs/src/convert.rs | 336 | 3 | - | done | sample normalization/linearize/CMYK; pure |
| crates/codecs/src/display.rs | 442 | 2 | - | done | display ICC profile parse + moxcms transforms; profile id = DefaultHasher of bytes (local cache key); catch_unwind guards. No I/O. |
| crates/codecs/src/exif.rs | 166 | 3 | - | done | minimal TIFF/EXIF reader (orientation, thumbnail offset) with bounds checks; minimal_exif builder |
| crates/codecs/src/gainmap.rs | 834 | 2 | - | done | ISO 21496/Ultra HDR gain map compute/apply/encode/read; XMP namespace URIs are data. Embeds only gain-map container metadata. |
| crates/codecs/src/heif.rs | 151 | 3 | - | done | thin wrapper over optional lightcraft-heif (known young dep, see doc.md) |
| crates/codecs/src/icc.rs | 337 | 2 | - | done | ICC parse/write via moxcms; fixed_date() ZEROES creation timestamp (removes info, privacy positive); align_tags clears MD5 id |
| crates/codecs/src/jpeg.rs | 453 | 3 | - | done | marker parse (EXIF/XMP/ICC/MPF/ISO gain map), zune/jpeg-decoder decode, embedded thumbnail; in-memory |
| crates/codecs/src/png_codec.rs | 170 | 3 | - | done | png decode + checked chunk walk |
| crates/codecs/src/psd.rs | 437 | 3 | - | done | PSD composite decode, PackBits, 8BIM resources with checked bounds |
| crates/codecs/src/space.rs | 342 | 3 | - | done | TRC / colour space math |
| crates/codecs/src/tiff_codec.rs | 242 | 3 | - | done | tiff crate decode from Cursor, limits from DecodeOptions |
| crates/codecs/examples/jpeg_bench.rs | 61 | test | - | done | dev example; optional argv image path read via image::open; synthetic otherwise |
| crates/codecs/tests/bench.rs | 88 | test | - | done | timing only |
| crates/codecs/tests/corpus.rs | 146 | test | D(read-only) | done | reads gitignored corpus/ (LIGHTCRAFT_CORPUS override) and macOS /System/Library/ColorSync/Profiles read-only to parse ICC; nothing written/sent. Justified test coverage. |
| crates/codecs/tests/fuzz.rs | 131 | test | - | done | proptest fuzzing; inline 70-byte GIF crash reproducer (starts GIF89a) fed only to decoder |
| crates/codecs/tests/header.rs | 366 | test | - | done | synthetic headers |
| crates/codecs/tests/heif.rs | 98 | test | - | done | synthetic HEIC via lightcraft_heif::testdata |
| crates/codecs/tests/roundtrip.rs | 637 | test | - | done | encode/decode roundtrips in memory |
| crates/color/src/cct.rs | 171 | 3 | - | done | CCT/tint <-> xy math |
| crates/color/src/lib.rs | 239 | 3 | - | done | Mat3, colour spaces, Bradford; forbid(unsafe) |
| crates/color/src/perceptual.rs | 176 | 3 | F(lazy OnceLock) | done | OkLab/HSV/HSL; OnceLock = lazy matrix cache, not a startup hook |
| crates/color/src/spline.rs | 177 | 3 | - | done | monotone spline / 1D LUT |
| crates/color/src/transfer.rs | 125 | 3 | F(lazy OnceLock) | done | sRGB/gamma/PQ/HLG; lazy LUTs |
| crates/geom/src/crop.rs | 143 | 3 | - | done | crop geometry math |
| crates/geom/src/crop_drag.rs | 483 | 3 | - | done | crop handle drag math, bisection |
| crates/geom/src/homography.rs | 124 | 3 | - | done | 3x3 homography |
| crates/geom/src/lib.rs | 468 | 3 | - | done | Point/Vec2/Rect/Affine/Orientation; forbid(unsafe) |
| crates/geom/src/real.rs | 229 | 3 | - | done | f64/interval Real trait (outward-rounded intervals) |
| crates/raster/src/blur.rs | 126 | 3 | - | done | box/gaussian blur |
| crates/raster/src/histogram.rs | 87 | 3 | - | done | histogram stats |
| crates/raster/src/lib.rs | 313 | 3 | - | done | Image<T>, par_rows (rayon), crop/orient; forbid(unsafe) |
| crates/raster/src/resample.rs | 212 | 3 | - | done | Lanczos/Mitchell/box resize |
| crates/raster/examples/blur_bench.rs | 42 | test | E | done | dev-only example: one `unsafe { libc::clock_gettime(CLOCK_PROCESS_CPUTIME_ID, &mut ts) }` to read process CPU time; libc is dev-dependency. Benign, documented with SAFETY comment; outside crates/sysmem but not shipped. Info. |
| crates/scenes/src/lib.rs | 239 | 3 | - | done | 24 procedural demo scene defs; "captured" dates are synthetic metadata strings (not compared to clock) |
| crates/scenes/src/noise.rs | 68 | 3 | - | done | deterministic integer hash noise (seeded, no RNG/clock) |
| crates/scenes/src/paint.rs | 680 | 3 | - | done | procedural painters; hex literals are colours via hex(0xRRGGBB) |
| crates/scenes/examples/contact_sheet.rs | 29 | test | - | done | dev example writes PNG to argv path (default contact.png) |
| crates/merge/src/lib.rs | 59 | 3 | - | done | module wiring, error type, progress/cancel callback; forbid(unsafe) |
| crates/merge/src/align.rs | 143 | 3 | - | done | MTB/feature alignment math (rayon) |
| crates/merge/src/features.rs | 312 | 3 | - | done | Harris corners + descriptors |
| crates/merge/src/frame.rs | 198 | 2 | - | done | decodes caller-supplied bytes via lightcraft_raw/codecs/meta; no file access |
| crates/merge/src/hdr.rs | 449 | 3 | - | done | HDR merge/deghost |
| crates/merge/src/linalg.rs | 247 | 3 | - | done | small linear algebra, Rodrigues |
| crates/merge/src/output.rs | 150 | 2 | - | done | write_linear_dng returns Vec<u8> via lightcraft_raw::write_dng (no fs) |
| crates/merge/src/pano/blend.rs | 300 | 3 | - | done | multiband blend |
| crates/merge/src/pano/camera.rs | 449 | 3 | - | done | bundle adjustment |
| crates/merge/src/pano/finish.rs | 210 | 3 | - | done | crop/boundary |
| crates/merge/src/pano/mod.rs | 461 | 3 | - | done | stitch orchestration |
| crates/merge/src/pano/project.rs | 83 | 3 | - | done | projections |
| crates/merge/src/ransac.rs | 291 | 3 | G?no | done | fixed-seed xorshift PRNG for RANSAC sampling - deterministic, not a trigger |
| crates/merge/src/synth.rs | 86 | 3 | - | done | synthetic bracket DNGs / pano views from procedural scenes (bytes in memory) |
| crates/merge/tests/hdr.rs | 163 | test | - | done | synthetic HDR merge tests |
| crates/merge/tests/pano.rs | 247 | test | - | done | synthetic pano tests |
| crates/pipeline/src/lib.rs | 648 | 2 | D(env LIGHTCRAFT_PROFILE) | done | render plan/orchestration; profiling prints timings to stderr only; forbid(unsafe) |
| crates/pipeline/src/auto.rs | 303 | 3 | - | done | auto tone statistics |
| crates/pipeline/src/colorops.rs | 458 | 3 | - | done | HSL/mixer/grading; lazy OnceLock table |
| crates/pipeline/src/cull.rs | 138 | 3 | - | done | focus/clipping/similarity stats (classical, no models, no I/O) |
| crates/pipeline/src/dust.rs | 128 | 3 | - | done | dust spot visualisation math |
| crates/pipeline/src/finish.rs | 795 | 3 | - | done | per-pixel stage params; grain hash constants |
| crates/pipeline/src/geometry.rs | 390 | 3 | - | done | frame/sample geometry |
| crates/pipeline/src/local.rs | 454 | 3 | - | done | guided filters, NR, WB |
| crates/pipeline/src/local_tone.rs | 62 | 3 | - | done | local tone mapping |
| crates/pipeline/src/lut.rs | 143 | 3 | - | done | .cube text parser (size capped 2..=256) + in-memory LUT map named `registry()` (a HashMap, NOT the Windows registry) |
| crates/pipeline/src/masks.rs | 870 | 3 | - | done | mask evaluation; Instant only in #[cfg(test)] perf assertion |
| crates/pipeline/src/optics.rs | 903 | 3 | - | done | lens/CA/vignette warp; static Mutex cache keyed by image fingerprint (sampled FNV hash, local only) |
| crates/pipeline/src/output.rs | 335 | 3 | - | done | output colour spaces |
| crates/pipeline/src/profiles.rs | 310 | 3 | - | done | built-in look profiles (own numbers) |
| crates/pipeline/src/redeye.rs | 310 | 3 | - | done | red-eye detection/correction |
| crates/pipeline/src/spots.rs | 471 | 3 | - | done | heal/clone spots |
| crates/pipeline/src/tone.rs | 412 | 3 | - | done | tone curve LUTs |
| crates/pipeline/src/transform.rs | 89 | 3 | - | done | transform helpers |
| crates/pipeline/src/upright.rs | 611 | 3 | - | done | line detection / vanishing points; static analysis cache |
| crates/pipeline/src/visualize.rs | 383 | 3 | - | done | overlays (clipping, masks) |
| crates/pipeline/src/tests.rs | 523 | test | - | done | unit tests, in memory |
| crates/pipeline/src/tests_geometry.rs | 489 | test | D(env SCENE) | done | env SCENE picks a demo scene name for a test |
| crates/pipeline/src/tests_hdr.rs | 107 | test | - | done | in memory |
| crates/pipeline/src/tests_local.rs | 109 | test | - | done | in memory |
| crates/pipeline/src/tests_window.rs | 343 | test | - | done | in memory |
| crates/pipeline/examples/develop_sheet.rs | 37 | test | - | done | dev example writes PNG to argv path (default develop.png) |

## Findings

No malicious or suspicious behaviour found in this chunk.

- Info: crates/raster/examples/blur_bench.rs:8-14 - the only `unsafe` in the chunk: `libc::clock_gettime(CLOCK_PROCESS_CPUTIME_ID, ..)` in a
  dev-only example (libc is a dev-dependency). Reads process CPU time; benign. Outside the project's "unsafe only in crates/sysmem" rule but not shipped.
- Info (checked, justified): crates/gpu/src/backend.rs:163-185 writes/removes a GPU-init crash-sentinel file at a caller-supplied path
  (apps/lightcraft/src/main.rs:107-111 => config_dir()/gpu-init.marker; disabled with LIGHTCRAFT_NO_PREFS; --memory sessions only read).
  Content: backend set + unix time. No other production file I/O in the chunk.
- Info (checked, justified): GPU adapter selection (ctx.rs:281-363, nn.rs:84-137) skips `DeviceType::Cpu` (software rasteriser) and falls
  back to the CPU pipeline. This is a performance decision with an identical-output CPU path, not sandbox/VM detection; adapter
  names/limits go only to log::info/error strings and the local `perf.gpu*` status. Nothing is reported off-machine.
- Positive: backend.rs:132-146 forces DX12's FXC compiler so wgpu never loads an arbitrary dxcompiler.dll found on PATH (#471);
  icc.rs:157-165 zeroes the ICC creation timestamp in exported profiles.
- Not security (robustness only, noted for completeness): pipeline/src/optics.rs:351 `img.data[.. % n.max(1)]` would index an empty
  Vec if called on a 0x0 image; gpu/src/ctx.rs:505,529 assert! in non-test code (inside catch_unwind).

Known issues from doc.md: none re-encountered in this chunk (heic-rs is referenced via codecs/heif.rs wrapper only).

## Conclusion

All 115 files reviewed. Verdict: clean. No network, process spawning, persistence, data collection, obfuscated payloads, time/date
triggers, VM/debugger checks, or build scripts. The production crates all carry #![forbid(unsafe_code)]; the only file write is the
GPU crash sentinel at the app's own config path. WGSL shaders only compute on bound buffers and their entry points match the host's list.
