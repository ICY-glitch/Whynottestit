# S15: dynamic run in a disguised sandbox (strace)

Date: 2026-10-12. Commit under test: 0b8279f2 (branch claude/repo-security-audit-5k5448).
Scratch dir: `$S=/tmp/claude-0/-home-user-Whynottestit/00a59143-3780-5405-b67d-b5298842c712/scratchpad/s15`.
Spec: code-audit-plan.md "S15 details".

## Method / log

### Build
- Host: 4 CPU, 15 GB RAM, root in a container, strace 6.8, Xvfb/xvfb-run present, no `ip`, no faketime.
- `cargo fetch --locked` (via proxy) then the build itself OFFLINE, traced for network + exec so build scripts
  (category J) are observed too:
  `env -u CRAFT_FONTS_DIR CARGO_TARGET_DIR=$S/target strace -f --seccomp-bpf -e trace=connect,bind,sendto,execve -o $S/build.strace cargo build --release --locked --offline -p lightcraft -p lightcraft-cli`

### Disguise (scripts kept in `$S/bin/`: `run.sh`, `inner.sh`, `lo_up.py`, `scen_*.sh`)
`run.sh` = `env -i PATH=… unshare --mount --uts --net --pid --cgroup --fork --mount-proc bash inner.sh <scenario>`.
`inner.sh` (root, pid 1 of the new pid ns) then:
- hostname `alex-laptop`; new net ns with only `lo`, brought up by `lo_up.py` (ioctl SIOCSIFFLAGS). Verified: connect to
  1.1.1.1 -> `ENETUNREACH`; bind on 127.0.0.1 works. No route table entries at all.
- bind-mounted fakes: `/proc/cpuinfo` (no `hypervisor` flag, model `13th Gen Intel Core i7-13700H`), `/proc/uptime`
  (`389214.37 …`, ~4.5 days), `/proc/version` + `/proc/sys/kernel/osrelease` (Ubuntu `6.8.0-45-generic`),
  `/proc/1/cgroup` (`0::/init.scope`); cgroup namespace makes `/proc/self/cgroup` all `/` (before: memory cgroup path
  `process_api/…/claude-code-bash` leaked).
- `/sys/class` replaced by a tmpfs re-binding every real class except `virtio-ports`, plus `/sys/class/dmi/id/*` with
  Dell XPS 15 9530 values (sys_vendor, product_name, board_*, bios_*, chassis_*, modalias).
- `/run/systemd` hidden by a tmpfs (it held `container` = `docker`); `/container_info.json` overlaid with /dev/null.
- `/etc/passwd` + `/etc/group` with UID/GID 1000 = `alex` (instead of `ubuntu`); `/etc/hosts` (the real one listed
  `runsc` and `api.anthropic.com`) and `/etc/hostname` replaced.
- `/home` = tmpfs containing only `/home/alex` (bind of `$S/home/alex`); `/tmp` = fresh tmpfs (hides the scratch dir);
  `/run/user/1000` (0700 alex). Binaries at `/opt/lightcraft`; trace output to `/run/s15out` (0700 root, alex can't
  read it); scripts at `/run/s15bin`.
- App runs as `alex` via `setpriv --reuid=1000 --regid=1000 --init-groups env -i HOME=/home/alex USER=alex LOGNAME=alex
  LANG=en_US.UTF-8 PATH=/usr/local/bin:/usr/bin:/bin SHELL=/bin/bash DISPLAY=:0 XDG_RUNTIME_DIR=/run/user/1000
  TERM=xterm-256color XDG_SESSION_TYPE=x11 XDG_CURRENT_DESKTOP=ubuntu:GNOME DESKTOP_SESSION=ubuntu XDG_CONFIG_DIRS
  XDG_DATA_DIRS` — no container/CI/proxy/token variables. cwd = /home/alex.
- Strace: `strace -f -tt -s 256 -e trace=network,process,file,write -o /run/s15out/<name>.strace setpriv … <app>`.

Bait home `/home/alex` (24 files, mtimes 2026-09-20): Pictures/2026/Holiday/IMG_2041..2043.jpg, Pictures/Phone/
PXL_*.jpg (x2) + Screenshot_2026-08-05.png (copies/resizes of docs/images), Documents/{tax-2025.txt,bank-details.txt,
wallet.dat}, Downloads/{setup.deb,invoice-0815.pdf}, .bash_history, .bashrc, .ssh/{id_rsa (fake),id_rsa.pub,known_hosts},
.mozilla/firefox/{profiles.ini,k3j9x2.default-release/{logins.json,key4.db,cookies.sqlite}},
.config/google-chrome/{Local State,Default/{Login Data,Cookies}}. All contain the marker `BAIT`.

Spoof results (checked as alex inside the sandbox): hostname, id/user, env, cpuinfo flag+model, uptime, /proc/version,
osrelease file, /proc/1/cgroup, /proc/self/cgroup, DMI, /etc/hosts, no internet — all OK. `systemd-detect-virt` went
from `docker` to `kvm`.
**Could NOT spoof:** the `cpuid` instruction (hypervisor bit, leaf 0x40000000 -> `kvm` per systemd-detect-virt);
`uname(2)` release (`6.18.44-fc-v114`); `/proc/self/mountinfo` (shows the tmpfs/bind overlays); `/proc/1/comm`
= `bash` (not systemd); `TracerPid` in `/proc/self/status` is non-zero under strace; virtio block device `/dev/vda` in
/proc/mounts; MAC/veth (no `ip`; there is no network device except lo). These are covered by the static S1 checks.

### Scenario scripts
- (a) `scen_a.sh`: `lightcraft-cli render ~/Pictures/2026/Holiday/IMG_2041.jpg -o ~/Pictures/export/IMG_2041-edit.jpg --set light.exposure=0.7 --set light.contrast=15`
- (b) `scen_b.sh`: `lightcraft-cli snapshot --demo --script /tmp/tour.jsonl -o ~/Pictures/export/snap.png --size 1280x800`
  (ui.set detail, ui.settle, ui.screenshot); `lightcraft-cli mcp --demo < mcp.in` (initialize, notifications/initialized,
  tools/list, tools/call query_photos, tools/call list_controls).
- (c) `scen_c.sh`: Xvfb :0 (root, untraced, -nolisten tcp); `lightcraft --library "~/Pictures/LightCraft Library" --control 18615`
  traced; driver `ctl.py` (python3 -I, root, same netns) sends `c1.jsonl` (ui.inspect, library.import of
  ~/Pictures/2026 + ~/Pictures/Phone, library.info, ui.set detail, develop.set light.exposure=0.8, ui.settle,
  ui.screenshot /tmp/s15-detail.png, app.export dir=~/Pictures/export jpg 1600), idle 630 s, `c2.jsonl` (library.info,
  grid, app.quit); relaunch, `c3.jsonl` (inspect, library.info, 45 s, app.quit).
- (d) `scen_d.sh`: libfaketime 0.9.10-2.1 (Ubuntu archive .deb, sha256 25fc8987…11e9, extracted, not installed),
  `FAKETIME=+500d` (clock ~Feb 2028): CLI render + desktop app ~3 min on the same library (develop.set, idle 150 s, quit).
- Build trace so far: no connect/bind/sendto at all (offline build); executables run = rustc, cc/linker, 47
  build-script-build binaries, and the `libc` crate's build script probing `emcc -dumpversion` along PATH (known
  upstream libc behaviour, emscripten version detection; ENOENT everywhere).
- Build finished OK in 9m47s. Final build-trace totals: **0** connect/bind/sendto; execve = rustc (478), cc/collect2/
  lld (86 each), 57 build-script-build runs (ahash anyhow async-io av-scenechange crc32fast crossbeam-* curve25519-dalek
  generic-array getrandom libc lightcraft lightcraft-engine lightcraft-ui-egui matrixmultiply naga num-bigint-dig paste
  proc-macro2 pulp quote rayon-core rfd rustix rustls serde serde_core serde_json smithay-client-toolkit thiserror
  unicode-general-category wayland-* wgpu wgpu-core wgpu-hal winit zerocopy zmij), `pkg-config --variable=libdir <x11 libs>`
  ×16 from x11-dl's build script, `emcc -dumpversion` from libc's. Nothing unexpected. (Note: the build env still had
  this container's token env vars; the trace shows no build script made any network attempt, so nothing could leave.)

## Observations

Analyser: `$S/bin/analyse.py` (python3 -I) parses each trace: network calls, non-thread clone/execve, opens with
write/create flags, mutating fs calls, hits on bait paths, VM/system-info paths, read opens under /home and /tmp.
Lines before the `execve("/opt/lightcraft/…")` belong to `setpriv`/`env` (getpwuid via nscd socket attempts, /etc/passwd,
/etc/group, /proc/<pid>/status) and are excluded from the app's behaviour below.

### (a) CLI render — DONE (exit 0, 1.1 s, wrote 3200×2000 JPEG)
- Network: none from the app (only setpriv's AF_UNIX nscd attempts before exec). No AF_INET/AF_INET6 socket at all.
- Processes: no execve after the app's own exec; 4 threads (clone3 CLONE_THREAD), 0 child processes.
- Writes: exactly one file created: `~/Pictures/export/.lightcraft-134-0.tmp` (O_CREAT|O_EXCL), then
  `rename(... .tmp -> ~/Pictures/export/IMG_2041-edit.jpg)`; `mkdir ~/Pictures/export` (EEXIST). Atomic export pattern.
  The JPEG written starts with JFIF + EXIF with software tag `LightCraft`.
- Reads under /home: the input JPEG, `~/Pictures` (stat), and the Vulkan loader's standard search dirs
  (~/.config/vulkan/*, ~/.local/share/vulkan/*, /etc/vulkan/*; libvulkan.so.1 dlopen'd = wgpu adapter probing).
- System info read by the app: `/proc/self/maps` (std main-thread stack guard), `/proc/meminfo` (memory budget),
  `/proc/self/cgroup` + `/sys/fs/cgroup/cpu/cpu.cfs_{quota,period}_us` ×3 (Rust std `available_parallelism`),
  `readlink /proc/self/exe` ×10. Not read: /proc/cpuinfo, /sys/class/dmi, /proc/1/*, /proc/uptime, mountinfo,
  osrelease/version, any bait file.

### (b) snapshot --demo and mcp --demo — DONE (both exit 0)
- snapshot: 3 requests OK, wrote `~/Pictures/export/snap.png` 1280×800 via `.lightcraft-135-0.tmp` + rename (only write).
  Network: none from the app. Processes: 0 children, 19 threads. Bait hits: 0. Reads under home: only
  `~/.config/lightcraft/{camera-profiles, denoise-models/settings.json, models/settings.json}` (ENOENT; its own config)
  and /tmp/tour.jsonl. `/proc/self/cgroup` + cgroup cpu quota read 1172× each (Rust std `available_parallelism`
  called per frame or per job — a perf quirk, not a security issue). `/sys/devices/system/cpu/online` once (CPU count).
  It also opened `/media/alex` (ENOENT), `/media`, `/mnt` and stat'ed `/mnt/<child>/DCIM|dcim` for each child
  (attach, sandboxing, skills, user-data — host dirs, not hidden by the sandbox). Source: `crates/engine/src/devices.rs`
  `mount_parents()` = camera/memory-card detection (DCF `DCIM` folder) for File > Import from Device, rescanned in a
  background thread "lightcraft-devices" when the cached list is >2 s old. Only directory listing + stat of DCIM; no file
  contents read. Justified (Info).
- mcp: initialize / tools/list (318 tools) / query_photos (demo photos, camera "Synthetic X2") / list_controls all
  answered. **Zero** writes, zero reads under /home or /tmp, no network, no child processes, 2 threads.

### (c) desktop app — setup notes
- First attempt (22:07) exited 101: panic in third-party `xkbcommon-dl` ("Library libxkbcommon-x11.so could not be
  loaded"), the container lacks that library. Trace of the failed run (`out/failed1/`): only wrote
  `~/.config/lightcraft/logs/lightcraft.log` (mkdir + log rotation renames) and the emergency panic log
  `/tmp/lightcraft-panics.log`; read X11 defaults (~/.Xdefaults, ~/.XCompose, ~/.Xresources), its own
  `~/.config/lightcraft/{ui.json,gpu-init.marker}`; connected only to the X server's unix socket; 0 bait hits.
- Second problem: no EGL/Vulkan driver -> "WGPU error: Failed to create surface"; on that error path the app tries to
  show an error dialog via rfd -> `zenity` (not installed). Expected UI behaviour.
- Fix (no system changes): extracted Ubuntu noble .debs into `$S/opt/lib` (bind-mounted at /opt/lightcraft/lib):
  libxkbcommon-x11-0 1.6.0-1build1 (sha256 3befe840…5a23), libxcb-xkb1 1.15-1ubuntu2 (53bc0a58…a53228),
  libegl1 1.7.0-1build1 (e549f777…4e57), libegl-mesa0 25.2.8-0ubuntu0.24.04.4 (2b8363cb…9419), libwayland-server0
  1.22.0-2.1build1 (edbfa4b6…dca), libxcb-dri2-0 1.15-1ubuntu2 (21543459…21ac). The app process therefore gets two
  extra env vars: `LD_LIBRARY_PATH=/opt/lightcraft/lib __EGL_VENDOR_LIBRARY_FILENAMES=/opt/lightcraft/egl/50_mesa.json`.
  Renders with wgpu GL backend on llvmpipe (software).
- A smoke test in throwaway dirs (`LIGHTCRAFT_CONFIG_DIR=/tmp/smoke/cfg`, library /tmp/smoke/lib) confirmed start-up;
  mesa wrote its shader cache to ~/.cache/mesa_shader_cache. Home was then reset (removed ~/.cache/mesa_shader_cache and
  ~/.config/lightcraft) so the real run is a first launch.
- Real run started 22:12:24. Phase 1 results: new library seeded with 24 demo photos, `library.import` imported 6
  (ids 25–30, 0 failed), develop.set OK, `ui.settle` is snapshot-only (unknown method here, harmless), screenshot OK,
  `app.export` wrote `~/Pictures/export/IMG_2041.jpg` 1600×1000. Idle 630 s from 22:12:47.
- Mishap, recorded for honesty: the first (failed) scenario-c sandbox was not fully killed (only the app and Xvfb were);
  its inner script kept going in its OWN namespaces (separate netns, so its control client could not reach the new app)
  and at 22:12:09 relaunched the app without the graphics libs (it failed: "Failed to open connection to X server",
  exit 1). That stray launch only rotated `~/.config/lightcraft/logs/lightcraft.log` -> `lightcraft.1.log` and wrote a
  new log (trace in `out/failed1/c_app2.strace`). So the real run's first launch found `~/.config/lightcraft/logs/`
  already present; the library folder did not exist yet. No other effect.
- Correction after reading the trace: the real run did `mkdir ~/.config/lightcraft` = 0 (it did not exist), i.e. my
  home reset happened after the stray 22:12:09 launch. The real run was a true first launch (config and library new).
- Stray launch extras (failed1/c_app2): libX11 fell back to TCP `connect 127.0.0.1:6000` (X display :0 over TCP, standard
  Xlib fallback when the unix socket is missing; loopback, refused) and the error path ran
  `execve zenity --no-markup --error --title "LightCraft could not start" --text "LightCraft could not open its window: …"`
  along PATH (ENOENT). Error dialog via rfd: expected.

### (c) desktop app — DONE (launch 1 22:12:24–22:23:18 incl. 630 s idle, quit exit 0; relaunch 22:23:18–22:24:06, exit 0)
Traces `out/c_app1.strace` (9507 lines), `out/c_app2.strace` (3531 lines); summaries `out/c_app{1,2}.summary.txt`.
- Network, launch 1: the only AF_INET socket is the requested control server: `bind(12, 127.0.0.1:18615)`, `listen`,
  2× `accept4` from 127.0.0.1 (my driver). 4544 `sendto` calls are all on accepted control fds 13/14 (JSON reply written
  in tiny pieces, e.g. `sendto(13, "{", 1, MSG_NOSIGNAL, NULL, 0)` — unbuffered, perf quirk only). AF_UNIX: X server
  (`@/tmp/.X11-unix/X0`) ×2, D-Bus session bus `/run/user/1000/bus` (ENOENT; desktop integration), setpriv's nscd.
  **No connect() to any non-loopback address, no DNS (no /etc/resolv.conf open, no UDP socket).** Same for relaunch.
- Processes: 0 execve after the app's own exec in both launches (44 / 29 threads, 0 child processes).
- Idle: **zero** traced syscalls between 22:12:47.365 and 22:23:17.451 (10.5 min) — no file, network, process or write
  activity at all while idle (the next event is the driver's accept4 for the quit request).
- Writes, launch 1 (87 opens-for-write; 130 mutating calls), all inside:
  - `~/Pictures/LightCraft Library/` (the `--library` folder): catalog.lock, catalog.lock.owner (JSON
    `{"pid":142,"host":"alex-laptop","program":"LightCraft","version":"0.6.0","since":…}` — the host name is why
    `/etc/hostname` is read), catalog.log (+ .tmp/rename), catalog.snap (.tmp/rename), prefs.json, view.json,
    thumbs/XX/<hash>.jpg (35 thumbnails, mkdir per shard). On quit: unlink catalog.lock.owner.
  - `~/.config/lightcraft/`: logs/lightcraft.log (+ rotation renames), ui.json (.tmp/rename), gpu-init.marker (created
    before GPU init, unlinked after success: crash marker).
  - `~/Pictures/export/.lightcraft-142-N.tmp` -> `IMG_2041.jpg` (the requested export) and `/tmp/.lightcraft-…tmp` ->
    `/tmp/s15-detail.png` (the requested screenshot).
  - `~/.cache/mesa_shader_cache/*` (written by the Mesa GL driver, not LightCraft code); `/dev/udmabuf` O_RDWR attempt
    (ENOENT; Mesa/wgpu buffer probe).
- Writes, relaunch: same library/config files; plus 34 `utimensat(fd, NULL, [UTIME_OMIT, now])` on thumbnails opened
  O_WRONLY without O_TRUNC (thumbnail-cache LRU mtime touch on cache hits; library info showed `hits: 26`).
- Originals: the 6 imported photos were opened read-only only (no O_WRONLY/O_RDWR/rename/unlink/chmod on any path under
  ~/Pictures/2026 or ~/Pictures/Phone in any trace); sha256 of the 5 JPEGs after all runs equals the docs/images sources.
- Reads under /home: imported photos, own library/config files, X11 defaults (~/.Xauthority, ~/.Xdefaults*,
  ~/.Xresources, ~/.XCompose, ~/.config/XCompose), Mesa ~/.drirc, Vulkan loader dirs. **Bait hits: 0** (no access of any
  kind to ~/.ssh, ~/.mozilla, ~/.config/google-chrome, ~/Documents, ~/Downloads, ~/.bash_history, ~/Desktop).
- System info: /proc/meminfo, /proc/self/maps, /proc/self/cgroup + cgroup cpu quota (155× launch 1, std
  available_parallelism), /sys/devices/system/cpu/{online,possible,cpuN/cpu_capacity} (Mesa, right after drirc),
  /etc/hostname (lock owner), /media + /mnt/* DCIM probe (camera cards). **Not read in any trace:** /proc/cpuinfo,
  /sys/class/dmi, /proc/1/*, /proc/uptime, /proc/self/status (app; only setpriv), mountinfo, /proc/version, osrelease,
  /.dockerenv, /run/systemd, container_info.json, /sys/hypervisor.

### (d) clock +500 days (libfaketime) — DONE
`FAKETIME=+500d` via `LD_PRELOAD=/opt/lightcraft/libfaketimeMT.so.1` (verified: `date` -> Thu Feb 24 2028; app log lines
stamped `2028-02-24T22:26:10Z`; catalog.lock.owner `since: 1835043970` = 2028-02-24). Strace timestamps stay real time.
- CLI render (`d_render.strace`, 284 lines): wrote `~/Pictures/export/future.jpg` via .tmp + rename; otherwise only
  libfaketime's own /dev/shm/faketime_shm_* and sem.* files. No network, no exec, 0 bait.
- Desktop app on the existing library (`d_app.strace`, 6730 lines; 22:26:10–22:28:42, exit 0): AF_INET only
  `bind 127.0.0.1:18616` + accept4 from the driver; 0 execve; writes only library/config (+ libfaketime /dev/shm);
  thumbnails mtime-touched; **zero traced syscalls between 22:26:15 and 22:28:42** (the 150 s idle); 0 bait; no VM-check
  paths. No behaviour change with the future date.

## Conclusion
Behaviour matched the expectations in the plan in every scenario:
- **Network:** across all 7 app traces the only AF_INET activity is the opt-in control server binding 127.0.0.1 on the
  requested port and accepting my loopback driver. No outbound connect, no DNS lookup, no UDP socket, no IPv6. (The net
  namespace had no route anyway; strace would have logged any attempt, and none was made.) Build (traced): 0 network calls.
- **Programs run:** none by the app in normal operation. Only on start-up failure: `zenity --error` to show "LightCraft
  could not start" (rfd), and libX11's TCP fallback to 127.0.0.1:6000. Build: rustc/cc/lld, build scripts,
  pkg-config (x11-dl), `emcc -dumpversion` (libc crate).
- **Files written/deleted:** only the library folder given with `--library`, `~/.config/lightcraft/` (logs, ui.json,
  gpu-init.marker), the requested export/screenshot paths (atomic .tmp + rename), `/tmp/lightcraft-panics.log` after a
  panic, Mesa's shader cache, libfaketime's shm. Deletions: its own catalog.lock.owner and gpu-init.marker, temp files.
  Original photos never opened for writing and byte-identical afterwards.
- **Data collection:** 0 accesses (open, stat, list) to any bait (.ssh, browser profiles, Documents, Downloads, shell
  history) in any trace. Reads outside the user's photos were its own config/library, X11/Mesa/Vulkan standard files,
  /proc/meminfo, /proc/self/{maps,cgroup}, cgroup CPU quota, CPU count/capacity, /etc/hostname (lock owner),
  /media and /mnt (DCIM probe for camera cards).
- **Anti-analysis:** none of /proc/cpuinfo, DMI, /proc/1/*, /proc/uptime, mountinfo, /proc/version, osrelease,
  container markers or /proc/self/status (TracerPid) were read by the app. Idle 10.5 min (and 2.5 min with clock +500 d):
  completely silent.
- **Limits:** cpuid hypervisor bit, uname(2) release, mountinfo overlays, pid-1 name and TracerPid could not be hidden,
  so a detector using those could in principle have noticed — but the app never read the file-based ones, and S1's
  static scan covers cpuid. Desktop run needed 6 extracted Ubuntu libs (EGL/xkbcommon-x11) via LD_LIBRARY_PATH; GPU was
  software llvmpipe; Linux only (Windows/macOS code paths not exercised). Model downloads / update checks were not
  triggered (not requested) — consistent with "no connections unless a model download is requested".

Verdict for S15: **clean**.

## Artefacts kept (in `$S/`)
`out/*.strace` (raw traces), `out/*.summary.txt` (analyser output), `out/failed1/` (failed first attempt + stray launch),
`bin/` (all scripts), `home/alex` (bait home after the runs), `build.strace` (build exec/network trace), `build.log`.
Deleted afterwards: `$S/target` (build output), `$S/opt` binaries and libs, `$S/dl` (.debs).
