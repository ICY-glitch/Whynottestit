# Coverage index

Every code file was assigned to exactly one chunk (`audit/chunks/<chunk>.txt`). The per-file table (path, lines, tier, capabilities seen, status, note) is in each chunk's notes file.

| Chunk | Files assigned | Files reviewed (agent) | Not fully reviewed | Verdict | Notes file |
|---|---|---|---|---|---|
| S15-dynamic-run | n/a | 4 | none | **clean** | [`audit/S15-dynamic-run.md`](S15-dynamic-run.md) |
| S1-sweeps-binaries | n/a | 15 | none | **clean** | [`audit/S1-sweeps-binaries.md`](S1-sweeps-binaries.md) |
| S14-dependencies | n/a | 137 | First-party build scripts (apps/lightcraft/build.rs, crates/engine/build.rs, crates/ui-egui/build.rs): outside S14's third-party scope; Long build scripts (ring 0.17.14, portable-atomic 1.15.0, libc 0.2.189): I searched these for risky calls and read around every hit and all their functions, but did not read every comment or data line; Large proc-macro crates (zerocopy-derive, serde_derive, jni-macros, pin-project-internal) were searched across all their non-test source for risky calls, not read line by line | **clean** | [`audit/S14-dependencies.md`](S14-dependencies.md) |
| R1-entry-network-tooling | 111 | 111 | none | **clean** | [`audit/R1-entry-network-tooling.md`](R1-entry-network-tooling.md) |
| R2-engine-a | 70 | 70 | none | **clean** | [`audit/R2-engine-a.md`](R2-engine-a.md) |
| R3-engine-b | 35 | 35 | none | **clean** | [`audit/R3-engine-b.md`](R3-engine-b.md) |
| R4-ui-a | 30 | 30 | none | **clean** | [`audit/R4-ui-a.md`](R4-ui-a.md) |
| R5-ui-b | 68 | 68 | none | **clean** | [`audit/R5-ui-b.md`](R5-ui-b.md) |
| R6-catalog-meta-ai | 100 | 100 | none | **clean** | [`audit/R6-catalog-meta-ai.md`](R6-catalog-meta-ai.md) |
| R7-raw-tiff | 58 | 58 | none | **clean** | [`audit/R7-raw-tiff.md`](R7-raw-tiff.md) |
| R8-compute | 115 | 115 | none | **clean** | [`audit/R8-compute.md`](R8-compute.md) |

S1/S14/S15 counts are sweep categories + binaries, third-party build scripts + proc-macro crates, and test scenarios, respectively.
