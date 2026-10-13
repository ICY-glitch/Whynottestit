# R6 catalog / meta / AI chunk — malware audit notes

Scope: 100 files listed in audit/chunks/R6-catalog-meta-ai.txt (catalog, denoise-core, denoise, develop, faces, heif, meta, preview).
Method: chunk-wide capability grep first, then per-file read. Tiers: 1 = touches A-L capabilities (every line read), 2 = ordinary logic (every line read), 3 = pure computation (proved no A-L, skimmed).

## Files

| path | lines | tier | capabilities | status | note |
|---|---|---|---|---|---|
| crates/catalog/Cargo.toml | 23 | 1 (J) | J | done | deps: develop, meta, serde, serde_json, thiserror, crc32fast, log, web-time. No net/process deps, no build.rs. |
| crates/denoise-core/Cargo.toml | 26 | 1 (J) | J | done | deps: raster, serde(_json), sha2, thiserror, miniz_oxide, rayon, web-time. No build.rs. |
| crates/denoise/Cargo.toml | 32 | 1 (J) | J | done | optional faer-core =0.17.1 (pure Rust GEMM; comment notes arch-specific asm in its deps - dependency matter for S14). No net deps, no build.rs. |
| crates/develop/Cargo.toml | 19 | 1 (J) | J | done | geom, color, serde, miniz_oxide. Clean. |
| crates/faces/Cargo.toml | 25 | 1 (J) | J | done | ndarray =0.17.2, matrixmultiply =0.3.11 pinned, sha2, serde. No net deps. |
| crates/heif/Cargo.toml | 26 | 1 (J) | J | done | heic-rs =0.1.1 pinned (young dep, already logged in doc.md). No build.rs. |
| crates/meta/Cargo.toml | 19 | 1 (J) | J | done | tiff, geom, serde, quick-xml. Clean. |
| crates/preview/Cargo.toml | 17 | 1 (J) | J | done | raster, siphasher, jpeg-encoder, zune-jpeg, log. Clean. |
| crates/catalog/examples/persist_bench.rs | 216 | 1 (C,D) | C, G(env) | done | Dev-only benchmark. `remove_dir_all(&dir)` on the CLI-supplied scratch dir (lines 78, 214), documented in the header as "wiped per size" - a footgun if pointed at a real folder, but explicit, not shipped. Env PERSIST_BENCH_NO_LOAD just skips a step. Synthetic data only. Info. |
| crates/catalog/src/dates.rs | 276 | 3 | none | done | Pure calendar math (Hinnant civil-from-days), grouping headers. No clock reads, no date-constant triggers (date constants only in tests / range bounds 0000-9999). |
| crates/catalog/src/folders.rs | 314 | 2 | none | done | Folder tree built from stored path strings only; explicitly touches no disk. Recursion bounded by MAX_DEPTH=64. |
| crates/catalog/src/journal.rs | 699 | 1 (C,F) | C (via Store: fixed names catalog.snap / catalog.log / catalog.log.damaged-N), F (one named snapshot thread, only on request) | done | Op-log + snapshot persistence. Writes only through Store trait to fixed file names; damaged log preserved not deleted. Env LIGHTCRAFT_PROFILE only toggles stderr timings. CRC-checked replay. Clean. |
| crates/catalog/src/keywords.rs | 1443 | 2 | none | done | Keyword tree / rename / merge / Lightroom keyword-list text parse+write (pure strings, returns text; no file I/O here). List-file nesting capped at MAX_LEVELS=64; tests in-memory only. Robustness note: keyword_tree `build` recursion depth equals the number of `|` levels in a photo keyword, which is not capped here (see findings, Low). |
| crates/catalog/src/lib.rs | 848 | 2 | none (forbid(unsafe_code)) | done | Op enum + Catalog::apply in-memory state machine. Ops never touch disk (SetFile/RemovePhoto docs + code confirm; file moves happen in engine, other chunk). Batch rollback on error. Nested Batch depth bounded by serde_json default recursion limit (128). Clean. |
| crates/catalog/src/local.rs | 308 | 2 | G (date-age policy, caller-supplied now) | done | "Forget" untouched Local browse records older than DEFAULT_FORGET_DAYS=30: emits Op::RemovePhoto on catalog records only; module docs + code confirm files/XMP never touched. Age compare uses caller-supplied `now`, not a hard-coded date. Scoped threads for fingerprinting. Justified, not a hidden trigger. |
| crates/catalog/src/lock.rs | 140 | 1 (C,D) | C (creates catalog.lock, writes/removes catalog.lock.owner in the library dir), D (env COMPUTERNAME/HOSTNAME, /etc/hostname, pid) | done | Single-instance library lock via std File::try_lock. Hostname+pid+version+timestamp written locally to catalog.lock.owner purely for the "already open in X" message; never sent anywhere. Removes only its own owner file on drop. Justified (Info). |
| crates/catalog/src/model.rs | 565 | 3 | none | done | Data records (Photo, Meta, Album, Stack) + pure helpers. `Stack::top()` indexes `photos[0]` (panic path if a hand-edited snapshot held an empty stack) but no production caller found; robustness only. |
| crates/catalog/src/query.rs | 609 | 3 | none | done | Filter/search/sort, lexical path normalisation (no FS access). Random sort is a deterministic splitmix64 seed, not a trigger. Minor robustness: date_groups slices `&d[..4]/[..7]/[..10]` by byte (would panic on a non-ASCII capture string of len>=10 from a hostile snapshot) - not malware. |
| crates/catalog/src/rules.rs | 1667 | 2 | G (reads system clock for "in the last N days" rules) | done | Smart-album rule engine. Clock read (line 592) only to evaluate user "in the last..." rules; `set_now` lets the engine override; wasm fallback constant "2026-01-01" is a no-clock default, not a trigger (it only changes rule matching). Loop/depth guards (MAX_ALBUM_DEPTH=32). Tests in-memory. Clean. |
| crates/catalog/src/safe_file.rs | 388 | 1 (C) | C (temp+fsync+rename writer, hard_link publish, removes only its own `.lightcraft-<pid>-<n>.tmp`) | done | Crash-safe writer library; target paths come from callers (engine). write_new never replaces an existing file. Fault-injection hooks are thread-local and only called from engine tests. Tests use std::env::temp_dir()/lc-safe-file-*. Clean. |
| crates/catalog/src/stacks.rs | 352 | 3 | none | done | Stack grouping ops + iso_seconds parser (uses get(), ascii-digit checked). In-memory only. |
| crates/catalog/src/store.rs | 233 | 1 (C) | C (create_dir_all library dir, write/rename/truncate named files inside it) | done | FsStore confined to `dir.join(name)` where names are journal constants. MemStore for tests. Clean. |
| crates/catalog/src/tests.rs | 675 | test (skim) | none | done | In-memory catalog/op tests (MemStore). No FS/net. |
| crates/catalog/src/tests_album_order.rs | 122 | test (skim) | none | done | In-memory. |
| crates/catalog/src/tests_background.rs | 339 | test (skim) | C (temp only), F (test threads) | done | Background snapshot tests; disk test uses std::env::temp_dir()/lc-journal-bg-<pid>, removed afterwards. |
| crates/catalog/src/tests_folder_records.rs | 199 | test (skim) | none | done | In-memory. |
| crates/catalog/src/tests_folders.rs | 404 | test (skim) | none | done | Path-string tests only (/net/..., //srv/share are strings, no network). |
| crates/catalog/src/tests_format_version.rs | 217 | test (skim) | none | done | In-memory format-version tests. |
| crates/catalog/src/tests_journal.rs | 378 | test (skim) | C (temp only) | done | temp_dir()/lc-journal-<pid>, lc-journal-stream-<pid>; removed afterwards. |
| crates/catalog/src/tests_local.rs | 267 | test (skim) | none | done | In-memory. |
| crates/catalog/src/tests_lock.rs | 93 | test (skim) | B (re-execs current test binary), C (temp only) | done | Spawns `current_exe()` with `--exact tests_lock::lock_holder_child --ignored` to hold a lock, then kills it; env LC_LOCK_CHILD_DIR passes the temp dir. Test-only, self re-exec, justified. |
| crates/catalog/src/tests_torn_append.rs | 223 | test (skim) | C (temp only) | done | temp_dir()/lc-torn-<pid>. |
| crates/denoise-core/src/archive.rs | 279 | 1 (C, H-parser) | C (writes `dest` via `<dest>.part` + rename, dest supplied by caller) | done | Minimal zip extractor for the .dtmodel download. Entry name only compared, never used as a path (no zip-slip). Bounded: EOCD search 64K, central dir <=1 MiB, entry <=256 MiB and <=max_bytes, deflate via miniz_oxide with limit, CRC-32 verified; zip64/encrypted refused. Tests temp-only; ignored test reads LC_DENOISE_DTMODEL file. Clean. |
| crates/denoise-core/src/bayer.rs | 191 | 3 | none | done | Bayer layout + tile packing; checked arithmetic, get() access. Pure. |
| crates/denoise-core/src/hash.rs | 46 | 2 | reads a caller-given model file | done | Streaming SHA-256 (sha2). Clean. |
| crates/denoise-core/src/lib.rs | 16 | 2 | none | done | forbid(unsafe_code), module list. |
| crates/denoise-core/src/licence.rs | 34 | 3 | none | done | Licence struct + valid_id (lowercase/digit first, [a-z0-9._-], <=64 chars: no path separators). |
| crates/denoise-core/src/known.rs | 128 | 1 (A-data) | A (URL constant only; no network code here) | done | Single known model (RawNIND, darktable-ai GitHub release URL, pinned sha256 + size), behind non-default feature `rawnind-model`; `all()` is empty by default. Download performed elsewhere on user request. Clean. |
| crates/denoise-core/src/manifest.rs | 217 | 2 (H-parser) | none | done | Denoise manifest JSON: size cap 64 KiB, strict validation (id charset, https-only info URLs, sha256 hex, tile/overlap ranges, finite gains). Pure data, no code/command fields. Tests in-memory. |
| crates/denoise-core/src/product.rs | 616 | 1 (C, H-parser) | C (writes cache file at caller path via `.tmp`+rename, create_dir_all of its parent) | done | Denoised-image cache format LCDN: header <=4096 B, sides <=131072, pixel cap, per-strip CRC + deflate with output limit, length checked; any problem = cache miss. Tests temp-only. Clean. |
| crates/denoise-core/src/run.rs | 589 | 3 | G-env (LIGHTCRAFT_PROFILE timings only) | done | Tiled denoise driver over a TileRunner; MAX_PIXELS 100M, alloc via try_reserve, finite checks, cancel flag. Test-only sleeps (jitter). Pure compute. |
| crates/denoise-core/src/tiles.rs | 100 | 3 | none | done | Tile starts/weights, bounded tile count. Pure. |
| crates/denoise/src/lib.rs | 37 | 2 | none | done | forbid(unsafe_code); cpu/runtime behind feature `runtime`; synthetic is doc(hidden) pub. |
| crates/denoise/src/onnx.rs | 369 | 1 (H-parser) | reads a caller-given .onnx | done | ONNX graph -> fixed whitelist of ops (Conv 1x1/3x3, ConvTranspose 2x2, MaxPool 2x2, two-input Concat, LeakyRelu, DepthToSpace 2 CRD); anything else Unsupported. Weights are float data only, size-capped (MAX_BYTES, MAX_WEIGHTS). A model is pure numeric data: no custom ops, no external data, no file writes, no code paths derived from it. Tests write synthetic models to temp_dir. Clean. |
| crates/denoise/src/onnx_proto.rs | 295 | 1 (H-parser) | none | done | Hand-written bounded protobuf decoder (no protoc/codegen, no recursion into subgraphs: attribute graph fields rejected as unknown). External tensor data (fields 13/14), sparse initializers and custom op domains explicitly refused, so an .onnx can never make the loader open another file. Varint, length, item and string caps. Clean. |
| crates/denoise/src/net.rs | 345 | 3 | none | done | Network description + validation (MAX_OPS 512, MAX_CHANNELS 4096, MAX_WEIGHTS 128M, finite weights, chaining shapes). Pure data. |
| crates/denoise/src/cpu.rs | 386 | 3 | G-env (LIGHTCRAFT_PROFILE) | done | Pure-Rust im2col+GEMM (faer, Parallelism::None) over a validated Net; every slice via get()/checked; 512 MiB working-memory cap; non-finite outputs rejected. No I/O. |
| crates/denoise/src/cpu/tests.rs | 151 | test (skim) | C (temp only) | done | Writes synthetic ONNX to temp_dir and removes it; ignored test reads LC_DENOISE_MODEL path. |
| crates/denoise/src/reference.rs | 133 | 3 | none | done | Slow plain-loop reference interpreter; only called from tests (cpu/tests.rs, gpu/nn.rs tests). Uses [] indexing on validated net (never-crash style issue only, test-only use). No I/O. |
| crates/denoise/src/runtime.rs | 433 | 2 | reads model at caller path | done | CpuRunner::load = validate manifest + onnx::read + NetRunner; self-test on a deterministic xorshift noise tile (not a trigger). Tests use temp_dir; ignored test reads LC_DENOISE_MODEL. Clean. |
| crates/denoise/src/synthetic.rs | 234 | 3 | none | done | Builds a tiny ONNX U-Net in memory with deterministic pseudo-random weights for tests (doc(hidden) pub). No I/O. |
| crates/denoise/src/synthetic_proto.rs | 44 | 3 | none | done | Protobuf varint/field writers for synthetic models. |
| crates/denoise/tests/real_model.rs | 250 | test (skim) | D/C (env-selected dirs, opt-in) | done | #[ignore] test; reads model + mosaics from LC_DENOISE_MODEL/LC_DENOISE_MOSAICS, optional PNG output to LC_DENOISE_OUT. No download, no deletes. |
| crates/develop/src/controls.rs | 511 | 3 | none | done | Static slider spec table + get/set by id via macro (field access only). Pure. |
| crates/develop/src/lib.rs | 266 | 3 | none | done | DevelopSettings helpers (JSON merge, FNV hash, section toggles). forbid(unsafe_code). Pure. |
| crates/develop/src/segmask.rs | 183 | 3 (H) | none | done | base64 here is a plain codec for SAM mask logits stored in settings (int8 -> deflate -> base64); decode capped (side<=1024, inflate limit side^2, <=4 detail patches). Justified data encoding, not obfuscation. |
| crates/develop/src/presets.rs | 301 | 3 | none | done | Preset groups, JSON extract/deep_merge/interpolate. No I/O. (deep_merge recursion bounded by serde_json depth of its input.) |
| crates/develop/src/settings.rs | 1109 | 3 | none | done | Serde data model for develop settings + Default impls and small predicates. Grep: no fs/env/net/process/unsafe. Pure data. |
| crates/develop/src/tests_process.rs | 162 | test (skim) | none | done | Serialisation/process-version tests with JSON literals. |
| crates/faces/src/lib.rs | 26 | 2 | none | done | forbid(unsafe_code), module list. |
| crates/faces/src/manifest.rs | 386 | 2 (H-parser) | none | done | Face-model manifest: 64 KiB cap, id is a safe folder name (no `.`/`..`/separators), https-only info URLs, bounded sizes/means/stds/thresholds, decoder whitelist. Pure data. |
| crates/faces/src/catalog.rs | 228 | 1 (A-data) | A (user-supplied download URLs, data only) | done | Parses the user-owned models/catalog.json (256 KiB, 50 entries): https-only URL, sha256+size required, embedder role only, cannot shadow built-in ids/hashes; download file name sanitised to [A-Za-z0-9._-], must end .onnx, not dotfile (no traversal). Downloading itself is elsewhere and per the doc only on user action after licence prompt. Clean. |
| crates/faces/src/known.rs | 223 | 1 (A-data) | A (URL constants only) | done | Known face models by sha256: YuNet + SFace (github.com/opencv/opencv_zoo raw at pinned commit 25f423d0...), AuraFace (huggingface.co/fal/AuraFace-v1 at pinned revision). Each with pinned sha256 + size; non-commercial models never offered. Data only; legitimate public sources. |
| crates/faces/src/hash.rs | 46 | 2 | reads caller-given file | done | Byte-identical to denoise-core/src/hash.rs (streaming SHA-256). |
| crates/faces/src/matching.rs | 128 | 3 | none | done | Cosine ranking of face embeddings + suggestion thresholds. Pure; suggestions only, never auto-applied. |
| crates/faces/src/align.rs | 273 | 3 | none | done | Similarity-transform face crop/warp, checked access, output <=4096. Pure. |
| crates/faces/src/graph.rs | 442 | 1 (H-parser) | none | done | Bounded ONNX graph reader (512 MiB, 20k nodes/tensors, 64M elements, names 4096, lists 1M). Custom op domains, external tensor data (13/14), sparse initializers, multiple graphs refused; sub-graph attributes parsed as `Other` (never executed). No file access (works on bytes). Clean. |
| crates/faces/src/graph/tests.rs | 117 | test (skim) | none | done | In-memory synthetic protobufs. |
| crates/faces/src/onnx.rs | 514 | 1 (H-parser) | reads caller-given .onnx (probe) | done | Streaming protobuf probe with fixed (non-recursive) nesting depth, 5M field cap, 4096-byte strings, bounded lists, length checks vs parent and file size; seeks past weights. Read-only. Clean. |
| crates/faces/src/net.rs | 570 | 3 (H) | G-env (LIGHTCRAFT_PROFILE) | done | Float32 CNN interpreter with fixed op whitelist (Conv, Relu, Sigmoid, Add/Sub/Mul, PRelu, BatchNorm, Gemm, Flatten, Dropout/Identity, GlobalAveragePool, MaxPool, Resize, Transpose, Reshape); no control-flow ops, no I/O. Tensor caps 64M elements, finite checks. |
| crates/faces/src/net/kernels.rs | 306 | 3 | none | done | Conv/GEMM (ndarray general_mat_mul)/broadcast kernels; checked storage. Pure. |
| crates/faces/src/net/shape.rs | 496 | 3 | none | done | Shape inference + attribute validation before run. Pure. |
| crates/faces/src/net/recognition_tests.rs | 337 | test (skim) | none | done | In-memory synthetic recogniser tests. |
| crates/faces/src/runtime.rs | 352 | 2 | reads caller-given .onnx | done | Embedder::load (validate manifest, read <=512 MiB, graph::load, Net::new, shape-check) + catch_unwind guard + deterministic self-test. Tests temp-only; ignored test reads LC_FACE_MODELS dir. Clean. |
| crates/faces/src/suggest.rs | 237 | 3 | none | done | Proposes a draft manifest for a dropped .onnx from its probed shapes; generated id sanitised ([a-z0-9-], custom- prefix). Ignored test lists LC_FACE_MODELS dir. Pure. |
| crates/faces/src/synthetic.rs | 250 | 3 | none | done | In-memory minimal ONNX builders for tests/self-check. No I/O. |
| crates/faces/src/yunet.rs | 370 | 3 | none (tests: env LC_YUNET_MODEL/LC_YUNET_REF read-only) | done | YuNet detector: letterbox, run Net, decode grids, NMS; candidate cap 4000, side cap 65536. Pure compute on caller bytes. |
| crates/heif/src/lib.rs | 407 | 2 (H-parser wrapper) | none | done | Wraps heic-rs (pinned 0.1.1, already logged) under catch_unwind; pixel limits checked before decode; returns plain data (pixels, ICC, EXIF, XMP). No I/O. Test byte array HEIC_RS_BOX_PANIC (72 B) decodes to an ftyp/meta/uuid box fuzz regression fixture, benign. |
| crates/heif/src/orient.rs | 178 | 3 | none | done | clap/irot/imir pixel transforms, checked crop math. Pure. |
| crates/heif/src/vui.rs | 427 | 3 (H-parser) | none | done | HEVC SPS/VUI bit reader up to matrix_coeffs; returns None on anything it cannot follow. Pure. |
| crates/heif/src/ycc.rs | 326 | 3 | none | done | YCbCr->RGB16 + chroma upsampling (rayon on native). Pure. |
| crates/heif/src/testdata.rs | 542 | test-support (feature `testdata`) | none | done | Writes synthetic lossless HEIF bytes in memory (PCM CTUs, minimal CABAC). No I/O, no embedded blobs. |
| crates/heif/src/tests_synthetic.rs | 176 | test (skim) | none | done | In-memory round-trip tests. |
| crates/meta/src/lib.rs | 341 | 2 | none | done | Metadata model + extract(bytes) (byte-slice only, no FS). forbid(unsafe_code) present but, unlike other crates, no `deny(clippy::unwrap_used, ...)` attribute (coding-standard gap, not malware). |
| crates/meta/src/container.rs | 331 | 2 (H-parser) | none | done | Locates Exif/XMP/ICC/IPTC blocks in JPEG/PNG/WebP/TIFF/RAF/CR3 byte slices. Bounds reviewed: direct indexing is guarded by prior length/starts_with checks; get() for file-derived offsets. URL strings are XMP APP1 signatures, not network. No I/O. |
| crates/meta/src/cr3.rs | 737 | 2 (H-parser) | none | done | ISO-BMFF box walker for CR3: box budget 4096, depth 12, all offsets via get()/checked_add. 16-byte constants are the public Canon (85c0b687-...) and XMP (be7acfcb-...) UUIDs, not obfuscation. Returns byte ranges only; no I/O. |
| crates/meta/src/datetime.rs | 215 | 3 | none | done | Exif/ISO date parsing with validation. No clock reads. Pure. |
| crates/meta/src/exif.rs | 353 | 2 (H-parser + writer) | none | done | Exif read via lightcraft-tiff; write_exif serialises only fields of the given Metadata (no hidden/extra identifying tags such as host/user). No I/O. |
| crates/meta/src/gpx.rs | 374 | 2 (H-parser) | none | done | GPX track parser on a string via quick-xml (no external entity / DTD resolution; no network), depth 64, 5M points. Pure. |
| crates/meta/src/iptc.rs | 124 | 2 (H-parser) | none | done | IPTC-IIM record parser; bounds checked (get / length checks before indexing; extended size <=4 bytes). Pure. |
| crates/meta/src/tags.rs | 331 | 3 | none | done | Tag-name table + value formatting for the Info panel. Pure. |
| crates/meta/src/xmp.rs | 1043 | 2 (H-parser + writer) | none | done | XMP parse via quick-xml: depth 64, 200k nodes, only predefined/numeric entities (no DTD/external entity expansion, no fetch). Writer emits only Metadata fields + lc:* (key charset-checked) with quick_xml escaping; standard xpacket id. Namespace URIs (incl. ns.lightcraft.app) are identifiers, never fetched. `expect("len > 1")`/`expect("root")` in parse_tree are infallible by construction (style only). |
| crates/meta/src/xmp_merge.rs | 403 | 2 | none | done | Merges a fresh XMP into an existing sidecar string, preserving foreign content; span edits checked for overlap and verified by re-parse. Returns a String; no file I/O here. |
| crates/meta/tests/robust.rs | 53 | test (skim) | none | done | Proptest fuzzing of parsers in memory. No FS/net. |
| crates/preview/src/disk.rs | 230 | 1 (C) | C (writes/deletes thumbnails in the cache dir) | done | Thumbnail disk cache `<dir>/<2hex>/<32hex>.jpg`. Pruning/clear only touch names matching `is_cache_file` (2-hex shard + 32-hex stem + .jpg/.tmpX) and regular files (symlinks skipped), so foreign files in a shared dir are never deleted. Unreadable entry at its own key path is deleted. Justified. |
| crates/preview/src/hash.rs | 61 | 3 (H checked) | none | done | SipHash-1-3 128-bit with fixed keys 0x4c696768_74437261 / 0x66742043_6f6e7465 = ASCII "LightCra"/"ft Conte" (a readable constant, not a hidden key). Cache-key hashing only. |
| crates/preview/src/lib.rs | 181 | 1 (C,F) | C (via DiskCache), F (one named writer thread per deferred put) | done | Memory LRU over optional DiskCache; generation guard; `clear` delegates to DiskCache::clear (cache-shaped files only); `retire` leaves disk alone. Clean. |
| crates/preview/src/lru.rs | 117 | 3 | none | done | Cost-bounded LRU; process-wide atomic tick counter used only for recency ordering (not a trigger). |
| crates/preview/src/pool.rs | 310 | 2 (F) | F (worker threads started on first submit, named lc-job-N) | done | Generic prioritized job pool running caller-supplied closures; bounded shutdown wait. No I/O of its own. |
| crates/preview/src/tests.rs | 353 | test (skim) | C (temp only) | done | Uses temp_dir()/lc-preview-*; includes a test that foreign files in a shared cache dir survive prune/clear. |

All 100 files on the chunk list are in the table above (checked by diffing the list against the table).

## Chunk-wide capability sweep (what was looked for and where it showed up)

- A, network: no socket, HTTP client, DNS or URL-fetch code anywhere in the chunk. URLs exist only as data: model sources and downloads in denoise-core/known.rs, faces/known.rs and faces/catalog.rs (https only, pinned to a commit or revision, sha256 and size required); XMP/RDF namespace identifiers; GPX/XMP test literals.
- B, running programs: no Command::new or shell in production code. The only process spawn is catalog/src/tests_lock.rs, which re-runs its own test binary (current_exe) as a lock-holder child.
- C, file writes and deletes. Each one is confined to:
  - the catalog library dir (store.rs and journal.rs: fixed names catalog.snap, catalog.log, *.tmp, catalog.log.damaged-N);
  - catalog.lock and catalog.lock.owner;
  - safe_file.rs temp files `.lightcraft-<pid>-<n>.tmp` next to caller-chosen targets (no-replace publish for write_new);
  - denoise cache / extraction targets supplied by the caller (`.tmp` / `.part` + rename);
  - the preview cache, which only deletes names shaped `<2hex>/<32hex>.jpg|.tmpX`, skips symlinks and leaves foreign files alone.

  "Forget Local records" deletes catalog records only, never files. Tests write only under std::env::temp_dir(). The persist_bench example wipes the scratch dir the user passes on the command line.
- D, data collection: only lock.rs reads COMPUTERNAME / HOSTNAME / /etc/hostname plus the pid, and writes them locally into catalog.lock.owner for the "already open in X" message. Nothing is sent anywhere. Other env vars: LIGHTCRAFT_PROFILE (stderr timings), and test-only LC_* / PERSIST_BENCH_* variables.
- E, low-level calls: none. Every crate root has #![forbid(unsafe_code)]. No extern, asm or FFI. faer-core and matrixmultiply are pinned dependencies (dependency review belongs to S14).
- F, code running at launch: no link_section, ctor or #[used]. Threads only on demand: the catalog-snapshot worker, the lc-preview-write writer, lc-job-N pool workers, scoped fingerprint threads and rayon. catch_unwind guards in heif and faces/runtime only turn panics into errors.
- G, hidden triggers: the only clock reads are rules.rs "in the last N days" smart-album rules and lock.rs `since`. The 30-day Local-record forgetting compares against a caller-supplied `now`. No date constants act as triggers: the wasm `now()` fallback "2026-01-01" is a no-clock default for rule matching only. Fixed PRNG seeds are used for deterministic self-test tiles only.
- H, obfuscation: none. Byte arrays found and what they are:
  - Canon and XMP CR3 UUIDs (public);
  - the heif fuzz-regression fixture (72 B, an ftyp/meta/uuid box);
  - SipHash keys that spell "LightCra"/"ft Conte";
  - base64 in segmask.rs, a plain codec for SAM mask logits;
  - the standard XMP xpacket id.

  No include_bytes!.
- I, persistence or privilege: none.
- J, build-time code: no build.rs in any of the 8 crates. Cargo manifests declare no build-time or network dependencies.
- K, code doing more than its stated purpose: none. Every parser (zip, LCDN cache, ONNX x3, CR3/ISO-BMFF, JPEG/PNG/WebP/TIFF containers, IPTC, XMP, GPX, HEVC VUI) works on byte slices or strings and returns data. None writes files or touches the network.
  - The ONNX readers whitelist their operators and refuse external tensor data (fields 13/14), custom op domains and sparse initializers. So a model file can only supply numbers: it cannot run code, open other files or write files.
  - The zip extractor only compares entry names and never uses them as paths (no zip-slip).
- L, sandbox / VM / debugger detection: none. No cpuinfo, DMI, cgroup, .dockerenv, hypervisor, ptrace, TracerPid or long sleeps. Sleeps exist only in tests.

## Findings

1. **Low (robustness, not malware), PLAUSIBLE.** Unbounded recursion on hierarchical keyword depth.
   - Where: crates/catalog/src/keywords.rs:284 (`fn build` inside `keyword_tree`), and also the `learn` recursion at :499.
   - Cause: the recursion depth equals the number of `|` levels in a photo keyword. Photo keywords come from the XMP `lr:hierarchicalSubject` and `dc:subject` of imported files and sidecars, via `keywords::from_file`, with no level cap. MAX_LEVELS=64 applies only to keyword-list text files.
   - Effect: a sidecar holding one keyword like `a|a|a|...` with hundreds of thousands of levels (fits inside the XMP 200k-node / depth-64 limits because it is a single text node) would overflow the stack when the keyword tree is built. That is an abort, not a catchable panic, and it would recur on every open while the record is in the catalog.
   - Not executed, reasoning only.
2. **Info.** crates/catalog/src/lock.rs:72-75,113-116 writes the computer name, pid, program name/version and open time to `<library>/catalog.lock.owner`. This is justified (the "already open on X" message) and stays local, but a library on a shared or synced folder exposes the host name to other users of that folder.
3. **Info.** crates/catalog/examples/persist_bench.rs:78,214 calls `remove_dir_all` on the directory given as the first CLI argument. This is documented and dev-only (examples don't ship), but it would wipe a real folder if pointed at one.
4. **Info (coding standard).** crates/meta/src/lib.rs lacks the `#![deny(clippy::unwrap_used, clippy::expect_used, ...)]` attribute that every other crate root has. crates/meta/src/xmp.rs parse_tree has `.expect("len > 1")` / `.expect("root")`; these cannot fail by construction. Separately, catalog/src/query.rs date_groups (`&d[..4]` etc.) and model.rs `Stack::top` (`photos[0]`) index directly. These could panic only on a hand-crafted catalog snapshot. None of this is malicious.

## Conclusion

Verdict for chunk R6 (catalog, preview, meta, develop, denoise-core, denoise, faces, heif): **clean**. No malware or virus behaviour:
- no network code;
- no process execution outside a self-re-exec test;
- no unsafe code or FFI;
- no persistence, hidden triggers, obfuscation or anti-analysis code.

File deletion is tightly scoped (cache-shaped names, own temp files, catalog records rather than files). Model files and manifests are pure data, so loading a model cannot run code or write files. The only finding of substance is a Low robustness issue (stack overflow from very deep hierarchical keywords). Known issues from doc.md (heic-rs young dependency) were seen again and not re-investigated.
