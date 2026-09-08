# SM-S931B / S931BXXUCZZI4 porting record (One UI 9 beta 2)

**Status: offline port complete, NOT yet hardware-validated.** Derived from the
exact ZZI4 boot.img kernel Image and its BTF; hardware validation pending the
device actually updating to `S931BXXUCZZI4`.

## 1. Firmware identity

| field | value |
| --- | --- |
| model | `SM-S931B` (Galaxy S25, Snapdragon 8 Elite, codename `pa1q`) |
| AP/PDA | `S931BXXUCZZI4` (update from `S931BXXUCZZHL`) |
| display build | `CP2A.260605.016.S931BXXUCZZI4` (Android 17, SDK 37) |
| security patch | `2026-08-05` |
| build fingerprint | `samsung/pa1qxxx/pa1q:17/CP2A.260605.016/S931BXXUCZZI4:user/release-keys` |
| kernel release | `6.6.127-android15-8-p33f4ffe-abogkiS931BXXUCZZI4-4k` |
| kernel size | `39115264` |
| kernel SHA-256 | `10fe338eb71ab5e88d1793e9de353aab2ae9e6c96ea6e69f018c9959ef31ca7c` |
| boot.img SHA-256 | `ea02238ea3c24d938cce872f098409253f8699cb6276d14c378c13f94f73f102` |
| BTF interval in Image | `[0x18aca6c, 0x1ecd673)` (6425607 bytes) — byte-identical to ZZHL |

## 2. Why a separate profile

Same GKI base (6.6.127) but a different GKI build (`paa4b906` -> `p33f4ffe`)
and a different kernel binary (SHA above), so the ZZHL payload's baked-in
offsets are not guaranteed. Symbol recovery was repeated for this exact Image
(vmlinux-to-elf, kallsyms base `0xffffffc080000000`).

## 3. Offset deltas vs pa1q-S931BXXUCZZHL

All 25 profile offsets were re-derived and compared. **18 are identical**,
including every load-bearing data symbol:

- identical: `call_usermodehelper_exec_work` `0x000d1278`, `worker_thread`
  `0x000d9b04`, `noop_llseek` `0x003cb7f8`, `copy_splice_read` `0x004190cc`,
  `configfs_read_iter` `0x004983d4`, `configfs_bin_write_iter` `0x00498900`,
  `anon_pipe_buf_ops` `0x01278148`, `ashmem_fops` `0x01437480`,
  `system_unbound_wq` `0x0233ac60`, `nfulnl_logger` `0x02342080`,
  `init_task` `0x0234e2c0`, `root_task_group` `0x0255df80`,
  `selinux_state` `0x025a0810`, random_table boot_id data ptr `0x024785c8`,
  `sysctl_bootid` `0x02683910`, `ashmem_misc.fops` `0x024bcd80`,
  `SLIDE_TRACEFS_WORKER_CALLER_OFF` `0x000d9ba0`, tracefs event id `109`.

Changed (7 + fingerprint):

| macro | ZZHL | ZZI4 | delta |
| --- | ---: | ---: | ---: |
| `ASHMEM_IOCTL_OFF` | `0x00d7e744` | `0x00d7e6c0` | `-0x84` |
| `ASHMEM_COMPAT_IOCTL_OFF` | `0x00d7ee00` | `0x00d7ed7c` | `-0x84` |
| `ASHMEM_MMAP_OFF` | `0x00d7ee54` | `0x00d7edd0` | `-0x84` |
| `ASHMEM_OPEN_OFF` | `0x00d7f074` | `0x00d7eff0` | `-0x84` |
| `ASHMEM_RELEASE_OFF` | `0x00d7f0fc` | `0x00d7f078` | `-0x84` |
| `ASHMEM_SHOW_FDINFO_OFF` | `0x00d7f188` | `0x00d7f104` | `-0x84` |
| `KMALLOC_CACHES_OFF` | `0x0180e538` | `0x0180e578` | `+0x40` |
| `SLIDE_NFULNL_LOGGER_NAME_OFF` | `0x01791639` | `0x01791638` | `-1` |

The ashmem text block moved as a unit (-0x84); `kmalloc_caches` gained one
pointer slot (+0x40); the `nfnetlink_log` name string shifted one byte.

## 4. Structural layouts

Unchanged — the kernel's BTF blob is byte-identical to ZZHL's
(SHA-256 `e13df32a16b5536c43897542b4dbc2c7082f2aefb91249bc94a06bfc5870950c`
for both), so every struct layout (rt_mutex_waiter, task_struct, file_operations,
page/slab, configfs_buffer, workqueue/pool/work, pipe_buffer, miscdevice,
selinux_state, ctl_table, mm_struct) is guaranteed identical.

## 5. P0 fingerprint

Regenerated from the ZZI4 Image (`Image[0x1f0000 - slide]`, 32 rows x 8
qwords, readback-verified). It differs from the ZZHL table in the rows whose
text regions moved (0x0b0000-0x130000 area); the slide-0x1f0000 row keeps the
same shape. The probe offset `0x1f0000` remains valid (all 32 rows readable).

## 6. Build

```sh
make TARGET=pa1q-S931BXXUCZZI4 release   # requires ANDROID_NDK_HOME
```

- artifact: `build/pa1q-S931BXXUCZZI4/cve-2026-43499-app.release.so`
- size: `97120` padded to `104128` (registry canonical size)
- SHA-256: `2f9799912ec6c297d9190bc2b82ff417ff451854a08e9524fa549743470284af`
- embedded label verified: `pa1q-S931BXXUCZZI4-app-physical-p0-oracle`

## 7. Scope

Offline only. No hardware run yet — requires flashing `S931BXXUCZZI4` and
running the full chain (exploit -> temp root -> KernelSU late-load) on device.
