# R7-raw-tiff — line-by-line audit notes

Scope: crates/raw (raw decoders, demosaic, DNG reader/writer, colour) and crates/tiff (TIFF/IFD reader/writer), 58 files, ~24.3k lines.
Question: does any of this behave like malware? Checklist A-L applied per file.

Chunk-wide pre-checks (done first):
- Capability grep over all 58 files (net, process, fs write/remove/rename, env, unsafe, extern, asm, link_section, used, include_bytes/str, time, rand, base64, cpuid/VM strings, ptrace, clipboard, registry, thread spawn, libc/windows/objc, dlopen, panic hooks, statics). Production `src/` hits: only `std::time::Instant` (timing debug/profiling) and `String::from_utf8_lossy` (TIFF ASCII tags). `std::env::var_os` + `std::fs::read` appear ONLY inside `#[cfg(test)]` / tests / examples (to be confirmed per file).
- No build.rs in crates/raw or crates/tiff; no proc macros; no macro_rules.
- `#![forbid(unsafe_code)]` at crates/raw/src/lib.rs:25 and crates/tiff/src/lib.rs:19.
- Trojan-Source scan (bidi overrides U+202A-E, U+2066-9, zero-width U+200B-F, BOM): none. All files valid UTF-8.
- Raw control bytes: only cr3.rs:446 and pef.rs:448 contain literal NUL bytes inside byte-string literals (`b"II*\0\x08..."`, `b"AOC\0MM"`) — test fixtures (checked below).
- Manifests: raw deps = sibling L0/L1 crates + serde, thiserror, rayon, miniz_oxide, zune-jpeg, zune-core, optional jxl-oxide; dev-deps jpeg-encoder, proptest, zune-jpegxl, sha2. tiff deps = serde, thiserror. No networking/process crates, no build-deps.

| path | lines | tier | capabilities | status | note |
|---|---|---|---|---|---|
| crates/raw/Cargo.toml | 35 | 1 | J(none) | done | pure-Rust decode deps only; no build script, no build-deps |
| crates/tiff/Cargo.toml | 16 | 1 | J(none) | done | serde + thiserror only |
| crates/raw/examples/colstats.rs | 17 | test-only (example) | D(argv file read) | done | reads argv[0] file, prints column stats. No writes. |
| crates/raw/examples/rawdump.rs | 14 | test-only (example) | C(write to argv[1]) | done | writes decoded samples to user-given OUT path only. Benign dev tool. |
| crates/raw/examples/rawinfo.rs | 133 | test-only (example) | D(argv), Instant | done | prints TIFF tree/decode stats/timings for given files; no writes. |
| crates/raw/examples/rawrender.rs | 64 | test-only (example) | C(write argv[1]) | done | develop -> 8-bit TIFF to user-given path. Benign. |
| crates/tiff/src/lib.rs | 467 | 3 | none | done | read every line: byte-order helpers, Value/Ifd types, accessors; `#![forbid(unsafe_code)]` + no-panic deny attrs; only std/serde. `from_utf8_lossy` = ASCII tag decoding. |
| crates/tiff/src/reader.rs | 238 | 3 | none | done | read every line: bounds-checked IFD parse, visited-set loop guard, depth/IFD/entry/value-byte budgets. Parses only the slice passed in. |
| crates/tiff/src/image.rs | 164 | 3 | none | done | read every line: strip/tile layout + chunk slicing of given buffer. |
| crates/tiff/src/writer.rs | 293 | 3 | none | done | read every line: serialises IFDs into an in-memory Vec<u8>; no file I/O (caller writes). |
| crates/tiff/src/makernote.rs | 202 | 3 | none | done | read every line: vendor maker-note header detection by magic prefix (Nikon/Fuji/Olympus/Panasonic/Pentax/Sony); `make` param unused (`let _ = make`). Tests in-memory. |
| crates/tiff/src/tags.rs | 226 | 3 | none | done | verified mechanically: every line is a `pub const ...: u16/u32 = N;`, comment, or `pub mod` wrapper; no code. |
| crates/tiff/tests/roundtrip.rs | 395 | test-only | none | done | skimmed: in-memory write/parse roundtrips + proptest fuzz; no fs/env/net. |
| crates/raw/src/lib.rs | 1182 | 3 | none | done | read lines 1-1000 fully, skimmed tests: probe() magic/TIFF heuristics, decode dispatch, RawImage/normalize/develop math. No I/O; tests build TIFFs in memory. |
| crates/raw/src/vendor/mod.rs | 119 | 3 | none | done | read all: Exif CFAPattern, black/white level estimation. |
| crates/raw/src/dng.rs | 430 | 3 | none | done | read prod code fully (deinterleave w/ factor cap 16, linearization, levels, CFA, colour tags, opcode lists, meta from_tiff). Tests skimmed. |
| crates/raw/src/tiffraw.rs | 380 | 3 | none | done | read all: chunk decode (none/ljpeg/lossy JPEG via zune-jpeg/deflate via miniz/JXL), predictors, alloc plausibility bound (2048 samples/byte). |
| crates/raw/src/unpack.rs | 223 | 3 | none | done | read all: bit unpacking, f16/f24, zlib inflate with output limit, predictor undo. |
| crates/raw/src/jxl.rs | 320 | 3 | none | done | read prod code: jxl-oxide decode from in-memory Cursor, AllocTracker memory cap, no thread pool. Colour matrix math only. |
| crates/raw/src/ljpeg.rs | 1319 | 3 | none | done | structural skim of all items + string literals, read decode() fully: T.81 lossless JPEG decoder/encoder, Hasselblad mode; bounded by max_samples and entropy-length checks. |
| crates/raw/src/preview.rs | 453 | 3 | none | done | read prod code fully: embedded JPEG/JXL preview extraction from the given byte buffer (TIFF tags, maker notes, CR3 boxes, bounded linear JPEG scan). Returns bytes to caller; no writes. |
| crates/raw/src/dngwrite.rs | 364 | 3 | none | done | read all: builds DNG IFD into Vec<u8> (Software="LightCraft", XMP from image metadata via lightcraft_meta::write_xmp, tiles via miniz/ljpeg). Returns bytes; caller decides path. No hidden tags/watermarks beyond Software/UniqueCameraModel fallback. |
| crates/raw/src/semantic.rs | 136 | 3 | none | done | read all: DNG semantic mask IFDs, capped at 16 masks / 2^25 samples. |
| crates/raw/src/opcodes.rs | 720 | 3 | none | done | read parser + apply: DNG opcode lists are a fixed enum of image transforms (warp, vignette, bad pixels, gain map, tables); unknown ids are kept as inert bytes, never executed. Tests skimmed. |
| crates/raw/src/spectral.rs | 171 | 3 | none | done | read prod code: make/model lookup into static colour-matrix table. |
| crates/raw/src/spectral_table.rs | 349 | 3 | H(checked) | done | large static table: every entry is `Camera { make, model, color_matrix: [Mat3..], forward_matrix: [Mat3..] }` of small floats (verified visually across the file) + ALIASES (make, alias, model) string triples. No encoded blobs. Header says generated from rawtoaces-data (Apache-2.0). |
| crates/raw/src/color.rs | 441 | 3 | none | done | imports only crate/lightcraft_color/serde; fn list = DNG colour model math (illuminant interpolation, WB, matrices, grey world). |
| crates/raw/src/profile.rs | 398 | 3 | none | done | imports only colour math/serde; HSV look tables + tone curve; MAX_ENTRIES cap. |
| crates/raw/src/gaintable.rs | 568 | 3 | none | done | ProfileGainTableMap parse/serialize/eval; caps MAX_TABLES/MAX_GAINS. `env::var_os("GTM_FILE")`+`fs::read` only in `#[cfg(test)] mod bench` #[ignore] test (line 549-567). |
| crates/raw/src/saturation.rs | 270 | 3 | none | done | clip-plateau detection on samples; constants are thresholds. |
| crates/raw/src/highlight.rs | 881 | 3 | none | done | highlight reconstruction (pyramid push/pull); `Instant` only in #[ignore] bench test at 848. |
| crates/raw/src/binned.rs | 252 | 3 | none | done | superpixel binning; rayon only. |
| crates/raw/src/demosaic/mod.rs | 253 | 3 | none | done | read prod code: method dispatch, reflect, mosaic/psnr helpers. `Instant` only in #[ignore] throughput test. |
| crates/raw/src/demosaic/bilinear.rs | 78 | 3 | none | done | imports Mosaic/Rgb32f/par_rows only; interpolation math. |
| crates/raw/src/demosaic/ppg.rs | 89 | 3 | none | done | same; PPG math. |
| crates/raw/src/demosaic/ahd.rs | 165 | 3 | none | done | same; AHD tiled math (TILE/MARGIN consts). |
| crates/raw/src/demosaic/xtrans.rs | 281 | 3 | none | done | same; X-Trans directional interpolation. |
| crates/raw/src/vendor/sr2.rs | 301 | 3 | H(justified) | done | read all. 127-word u32 SEED + XOR keystream `k[n]=k[n-63]^k[n-127]` decrypts Sony's encrypted SR2SubIFD (key 11 22 33 44) from the opened ARW; output is used only by `shorts()` to read black level / WB_RGGBLevels SHORT tags. This recurrence matches the publicly documented Sony SR2 cipher structure; it is a format decryption, not a hidden payload (nothing is decrypted from the binary itself; SEED is only XOR-ed against file bytes). Tests use sha2 only to pin a known-answer digest. |
| crates/raw/src/vendor/srw.rs | 541 | 3 | none | done | read prod code fully: Samsung SRW uncompressed; "EncryptionKey" is just an additive per-element offset on maker-note vectors. |
| crates/raw/src/vendor/arw.rs | 1321 | 3 | H(justified) | done | read prod code (1-711) fully: ARW2 block decode, packed12, LJ92 quad tiles, YCbCr M/S tiles. `DECIPHER` = inverse of Sony maker-note byte substitution `p^3 mod 249` (const-evaluated table), applied to 6 bytes of maker-note Tag2010 for WB only. Model lists are camera names for WB offsets. Tests skimmed. |
| crates/raw/src/vendor/arw_lens.rs | 230 | 3 | G(checked, benign) | done | read prod code fully: model string compare `model != "ILCE-7RM4A"` only gates a lens-distortion warp polynomial fit; output is an Opcode, no side effects. |
| crates/raw/src/vendor/cr2.rs | 391 | 3 | none | done | read prod code fully (1-222): IFD3 LJ92 strip, slice reassembly (slice count capped 64), maker-note SensorInfo/ColorBalance, black from masked columns. |
| crates/raw/src/vendor/cr2/sraw.rs | 578 | 3 | none | done | outline + read decode/read_preview: sRAW YCbCr frames; chroma gains regressed against the same file's IFD2 16-bit preview (n capped 2^24). |
| crates/raw/src/vendor/cr3.rs | 550 | 3 | none | done | outline + read decode(): ISO-BMFF via lightcraft_meta::cr3, CRX decode, maker-note levels. Line 446 (test) holds raw NUL/control bytes inside `b"II*..."` literal = TIFF header `II*\0\x08\0\0\0\x01\0` written with literal bytes instead of escapes; harmless test fixture (file flagged "data" by `file`). |
| crates/raw/src/vendor/crx.rs | 1096 | 3 | none | done | outline of all items + every string literal (all error messages) + imports (crate, meta::cr3, rayon): Canon CRX Rice/wavelet decoder with caps (MAX_TILES, MAX_HEADER_BYTES, MAX_CRX_SAMPLES). Test at 1064 reads optional `LIGHTCRAFT_CR3_CORPUS` dir (test-only, read-only). |
| crates/raw/src/vendor/crx_wavelet.rs | 338 | 3 | none | done | outline: inverse 5/3-style wavelet lifting with overflow checks; only imports crate error types. |
| crates/raw/src/vendor/nef.rs | 391 | 3 | none | done | read prod code (decode/uncompressed/compressed) fully: NEF uncompressed/Huffman dispatch, JPEG XS (HE) refused with message. |
| crates/raw/src/vendor/nefc.rs | 710 | 3 | none | done | outline + strings: Nikon Huffman/lossy decode; constant Huffman tables (counts/symbols, small) and fixed-rate word tables are codec data. |
| crates/raw/src/vendor/pef.rs | 491 | 3 | none | done | read decode() fully + outline: Pentax Huffman/packed. Line 448 (test) has a literal NUL byte in `b"AOC\0MM"` maker-note magic; harmless fixture. |
| crates/raw/src/vendor/orf.rs | 655 | 3 | none | done | outline + read preview(): Olympus ORF uncompressed variants, maker-note sub-IFDs, CFA from data; compressed ORF refused. |
| crates/raw/src/vendor/raf.rs | 387 | 3 | none | done | outline + read header(): FUJIFILMCCD-RAW header offsets (bounded `get`), record dir capped 4096. |
| crates/raw/src/vendor/rafc.rs | 505 | 3 | none | done | outline + strings: Fujifilm compressed RAF entropy decoder; constant XTRANS/MASKS tables are small layout data. |
| crates/raw/src/vendor/rw2.rs | 1312 | 3 | none | done | outline + strings + read decode() start and module doc: Panasonic RW2 words/blocks/format-8 Huffman strips, distortion tag -> warp opcode. Caps MAX_SAMPLES, MAX_STRIPS=64. |
| crates/raw/tests/corpus.rs | 933 | test-only | D(read corpus dir) | done | skimmed: reads `$LIGHTCRAFT_CORPUS` or `../../corpus` (read_dir/read only), `include_str!("docs/raf-corpus.sha256")` = 64-hex SHA-256 + path lines used to verify corpus identity. No downloads, writes, deletes, processes. |
| crates/raw/tests/cr3_corpus.rs | 93 | test-only | D(read corpus) | done | same pattern, read-only, skips when files absent. |
| crates/raw/tests/dng.rs | 991 | test-only | none | done | skimmed: synthetic DNG round trips in memory via TiffWriter/write_dng/miniz; no fs. |
| crates/raw/tests/robust.rs | 104 | test-only | none | done | proptest fuzzing of decode/ljpeg on random bytes; Instant used for a time-bound assert. |
| crates/raw/tests/vendor_robust.rs | 303 | test-only | none | done | synthetic vendor files mutated/truncated in memory; no fs. |

## Findings

No Critical / High / Medium findings. Production code in crates/raw/src and crates/tiff/src has **no** network, process, filesystem, environment, clipboard, OS/FFI, unsafe (`#![forbid(unsafe_code)]` in both crate roots), threads-at-launch, static initialisers, build scripts or time/random triggers. Every decoder takes `&[u8]` and returns data structures; the only writer (dngwrite / tiff::writer) returns `Vec<u8>` to the caller.

Info-level notes (all justified, recorded for completeness):
1. **XOR keystream in sr2.rs (H, justified)** — `SEED: [u32; 127]` + recurrence `k[n]=k[n-63]^k[n-127]` (sr2.rs:52-87) decrypts Sony's encrypted SR2SubIFD from the user's own ARW, only to read black-level / WB SHORT tags. Matches the publicly known structure of Sony's SR2 cipher. Not applied to any embedded blob.
2. **Byte-substitution table in arw.rs (H, justified)** — `DECIPHER` (arw.rs:89-98) is the const-evaluated inverse of `p^3 mod 249`, applied to 6 bytes of the Tag2010 maker-note block for WB fallback.
3. **Literal NUL bytes in test byte-string literals** — cr3.rs:446 (`b"II*<NUL><BS>..."`) and pef.rs:448 (`b"AOC<NUL>MM"`). Harmless test fixtures, but they make `file`/`grep` treat these .rs files as binary ("data"), which can make naive grep-based review skip them (needed `grep -a`). Suggest escaping as `\0`/`\x08`.
4. Model-name gates (arw_lens.rs `"ILCE-7RM4A"`, arw.rs TAG2010_WB model lists, srw/nef/orf constants) only select decoding parameters; no hidden behaviour.

## Conclusion

Verdict for R7-raw-tiff: **clean**. 58/58 files reviewed (Tier-3 codec/math files proven capability-free by grep + imports and skimmed structurally with entry points read in full; TIFF crate and lib/dng/tiffraw/unpack/preview/dngwrite/opcodes/sr2/arw/srw read line by line). Nothing behaves like malware; the code matches its stated purpose of decoding raw/TIFF bytes it is handed.
