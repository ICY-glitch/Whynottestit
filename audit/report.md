# Final report: is LightCraft malware?

**Date:** 2026-10-13 · **Code audited:** commit `0ba118de` (identical to public `storytold/lightcraft` main) ·
**Method:** `code-audit-plan.md`, run as a parallel-agent workflow (`wf_573e1d86-7f2`, 14 agents)

## Verdict

**No. LightCraft is not malware.** Nothing was found in any part of the audit that spies on the user,
steals or sends data, gives anyone remote access, damages files, hides from analysis, or persists on the system.

The evidence comes from three independent angles that all agree:

1. **Reading the code.** All **587 code files** (about 207,000 lines) were read by eight reviewer agents against a
   12-point malware checklist (`code-audit-plan.md`). A cross-check confirmed every assigned file appears in its
   chunk's notes. Every chunk's verdict: **clean**.
2. **Watching it run.** The Linux build ran in a sandbox disguised as a normal Dell laptop, with a fake user, a
   lived-in home, and bait SSH keys, browser profiles and documents. It was traced with `strace` across 7 runs:
   - **No outbound network** at all (only the opt-in loopback control port).
   - **No programs started.**
   - **0 accesses** to any bait file.
   - **No VM checks.**
   - Writes only to its library, settings and export folders. Original photos were byte-for-byte unchanged.
   - Nothing changed after 10.5 minutes idle or with the clock moved 500 days ahead.
3. **Checking what ships.** The official v0.6.0 binaries match their published checksums. They contain no
   anti-debugging or VM-detection imports and no unexplained hostnames. In the third-party dependencies, build
   scripts and proc-macros do only normal build work. No dependency is malicious or yanked.

## What was found (none malicious)

42 findings in total: **0 Critical, 0 High, 0 Medium**, 13 Low, and the rest Info (behaviour checked and found
justified). The three findings first rated Medium were each **downgraded to Low** by an independent verifier. Full list
with evidence: [`findings.md`](findings.md).

Most useful Low findings, all ordinary robustness or hardening bugs:

| Area | Finding |
|---|---|
| Automation channel (opt-in `--control` / MCP `--connect`) | A client can run any program via "Edit in External Editor" (`app` parameter), read the clipboard, and export/overwrite files at any path. The channel is off by default and localhost-only, so the risk is another program already running on your PC. |
| Export commands | `keyword.export`, `preset.export` and `curve.exportPresets` write to any path without the "don't overwrite originals" guard other exports use. `folder.rename`/`folder.move` accept any absolute folder. |
| Windows | `cmd /C start` with an unquoted path when opening a photo in the default app (from the first audit, `doc.md`). |
| Malformed-file robustness | Deeply nested XMP keywords can overflow the stack (crash). Preset zip import trusts the declared size (memory). A quick-xml advisory (RUSTSEC-2026-0194, slow on crafted XMP). These are crash/slowdown bugs, not code execution. |
| Downloads | The model downloader follows https→http redirects. SAM 3's tokenizer files aren't hash-pinned. TLS uses an alpha crypto provider. |
| Release tooling | The Windows signing step downloads an unpinned NuGet package. CI tool downloads aren't checksum-pinned. |

Worth reporting to the LightCraft developers. **None of it is evidence of malicious intent.**

## Why antivirus tools flag it (summary of the earlier follow-ups in `doc.md`)

- **"Keylogger":** comes from keyboard functions in the standard `winit` windowing library. The app only receives keys
  while its own window is focused, and never stores or sends them. Confirmed by code and by the traced run.
- **"Root certificate added":** Windows itself installs Microsoft's root certificate when it checks the app's
  Microsoft Trusted Signing signature. The exe has no certificate functions at all.

## Limits

- Only the **Linux** build was run; the Windows and macOS builds were checked statically (imports, strings, signature).
  `cpuid`, `uname`, `mountinfo` and `TracerPid` couldn't be disguised, but no code reads them.
- Optional AI-model downloads weren't triggered in the run (their code was read).
- Three long dependency build scripts (`ring`, `portable-atomic`, `libc`) and four large proc-macro crates were
  searched and read around every hit rather than line by line.
- Agents can make mistakes. The three-way agreement (code, run, binaries) and the independent verification of every
  Medium-or-worse finding are what make the verdict reliable.

## Files

`code-audit-plan.md` (plan) · [`files.md`](files.md) (coverage) · [`findings.md`](findings.md) (all findings) ·
`audit/<chunk>.md` (per-chunk notes with a per-file table) · `audit/chunks/*.txt` (file assignment) · `doc.md` (first
audit and follow-ups)
