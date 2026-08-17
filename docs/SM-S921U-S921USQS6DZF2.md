# SM-S921U / S921USQS6DZF2 porting record

This file records the evidence-driven port of the Galaxy S24 US (Qualcomm
Snapdragon 8 Gen 3) firmware. All values were derived from the actual
S921USQS6DZF2 firmware; no values were copied from the e3q (S928U) port.

## Stage 1: Firmware identity & inputs

Status: **COMPLETE**

| Field | Verified value |
| --- | --- |
| Package model | `SM-S921U` / `SM-S921U1` |
| AP/PDA package | `S921USQS6DZF2` |
| CSC | `S921UOYN6DZF2` |
| Device codename | `e1q` |
| Product name | `e1qsqw` |
| Platform / SoC | `pineapple` (Qualcomm Snapdragon 8 Gen 3, SM8650) |
| Android build ID | `UP1A.231005.007` |
| Build fingerprint | `samsung/e1qsqw/e1q:14/UP1A.231005.007/S921USQS6DZF2:user/release-keys` |
| Kernel release | `6.1.145-android14-11-33419968-abS921USQS6DZF2` |
| Page size | 4096 (4 KiB) |

### Extracted image hashes

| Object | Size (bytes) | SHA-256 |
| --- | ---: | --- |
| Decompressed `boot.img` | 100,663,296 | `2d7041a156862981ca5af5529c8fca4e8309c8e9fa5f1557491002336d3d7e55` |
| Raw kernel payload | 38,005,248 | `7b1a1c85ee54bc0ce867642be83f2c1b16252552310976f6a09b353846c06144` |
| Recovered `vmlinux.elf` | 43,070,883 | `6ca89d7905500bf9106f92ba38b29deae0fbfb90ab876a996c11c9b318ce5be7` |
| Extracted `vmlinux.btf` | 5,981,643 | `8415104c012e18942b18bcb52f401075cb6b92df837b9552a8c11070d65efe56` |

The raw BTF blob was located at kernel image interval `[0x180b384, 0x1dbf94f)`.

## Stage 2: Physical load address

Status: **COMPLETE**

Both e1q and e3q use the Qualcomm ABL with the same Snapdragon 8 Gen 3 memory
map. The DTB `/memory` node anchors `gunyah_hyp_region@80000000`. The ARM64
Image has `text_offset = 0` so the kernel lands at `0x80000000 + 0x00080000`:

```c
#define P0_PHYS_OFFSET      0x80000000ULL
#define P0_KERNEL_PHYS_LOAD 0x80080000ULL
```

## Stage 3: Symbol and BTF extraction

Status: **COMPLETE — derived from S921USQS6DZF2 vmlinux.elf**

Image base: `0xffffffc008000000`

| Macro | Symbol | Offset |
| --- | --- | ---: |
| `CALL_USERMODEHELPER_EXEC_WORK_OFF` | `call_usermodehelper_exec_work` | `0x000d39cc` |
| `SLIDE_TRACEFS_WORKER_CALLER_OFF` | `worker_thread` (schedule successor) | `0x000db1a0` |
| `NOOP_LLSEEK_OFF` | `noop_llseek` | `0x003a14e4` |
| `COPY_SPLICE_READ_OFF` | `generic_file_splice_read` | `0x003ef340` |
| `CONFIGFS_READ_ITER_OFF` | `configfs_read_iter` | `0x004712a4` |
| `CONFIGFS_BIN_WRITE_ITER_OFF` | `configfs_bin_write_iter` | `0x004717d4` |
| `ASHMEM_IOCTL_OFF` | `ashmem_ioctl` | `0x00d3a314` |
| `ASHMEM_COMPAT_IOCTL_OFF` | `compat_ashmem_ioctl` | `0x00d3ac4c` |
| `ASHMEM_MMAP_OFF` | `ashmem_mmap` | `0x00d3aca4` |
| `ASHMEM_OPEN_OFF` | `ashmem_open` | `0x00d3aed0` |
| `ASHMEM_RELEASE_OFF` | `ashmem_release` | `0x00d3af58` |
| `ASHMEM_SHOW_FDINFO_OFF` | `ashmem_show_fdinfo` | `0x00d3b078` |
| `ANON_PIPE_BUF_OPS_OFF` | `anon_pipe_buf_ops` | `0x01219d90` |
| `ASHMEM_FOPS_OFF` | `ashmem_fops` | `0x013d1140` |
| `SLIDE_NFULNL_LOGGER_NAME_OFF` | `"nfnetlink_log"` string (logger→name target) | `0x016a622a` |
| `KMALLOC_CACHES_OFF` | `kmalloc_caches` | `0x0176c6f8` |
| `SYSTEM_UNBOUND_WQ_OFF` | `system_unbound_wq` | `0x0223ae60` |
| `SLIDE_NFULNL_LOGGER_OBJECT_OFF` | `nfulnl_logger` object | `0x02242a20` |
| `INIT_TASK_OFF` | `init_task` | `0x0224f8c0` |
| `ASHMEM_MISC_FOPS_OFF` | `ashmem_miscs + 0x10` | `0x023bb5b0` |
| `SLIDE_RANDOM_TABLE_BOOT_ID_DATA_PTR_OFF` | `random_table[boot_id].data` | `0x023762f0` |
| `ROOT_TASK_GROUP_OFF` | `root_task_group` | `0x0244cd80` |
| `SELINUX_ENFORCING_OFF` | `selinux_state.enforcing` | `0x02521588` |
| `SYSCTL_BOOTID_OFF` | `sysctl_bootid` | `0x026046e8` |

`SLIDE_TRACEFS_EVENT_ID`: ftrace section base 20 + event index 86 = **106**.

## Stage 4: Target layouts & allocator

```text
sizeof(struct rt_mutex_waiter) = 0x58  (COMPACT_RT_MUTEX_WAITER)
sizeof(struct file_operations) = 0x110
sizeof(struct task_struct)     = 0x12c0
sizeof(struct page)            = 0x40
struct slab.slab_cache offset  = 0x18
```

All offsets verified against the S921USQS6DZF2 BTF.

## Stage 5: Slide & P0 configuration

```c
#define SKB_DATA_DELTA              (-0x1000LL)
#define SLIDE_PSELECT_WORD_SHIFT    3
#define APP_P0_FINGERPRINT_INVERSE_SLIDE 1
```

`p0_fingerprint.h` was generated from the S921USQS6DZF2 raw kernel at probe
offset `0x1f0000` with inverse-slide keying. Each row key is the physical
source offset `(probe - slide)`; the runtime converts it back to a slide.

## Stage 6: Payload build

Status: **COMPLETE**

```sh
make TARGET=e1q-S921USQS6DZF2 \
  ANDROID_NDK_HOME=/path/to/android-ndk-r29 release
```

| File | Size (bytes) | SHA-256 |
| --- | ---: | --- |
| `artifacts/e1q-S921USQS6DZF2/cve-2026-43499-app.so` | 104,128 | `6268f990e1decae6e0deda2c3cbb7be97dc63bb79c02894f3baf12a1e715dc1f` |

## Stage 7: KernelSU late-load artifacts

Status: **COMPLETE**

The `android14-6.1_kernelsu-e1q-S921USQS6DZF2-kdp.ko` module was rebuilt using
the exact e1q kernel release string:

```text
6.1.145-android14-11-33419968-abS921USQS6DZF2
```

It was audited against `/home/evo/devl/vmlinux.elf`, finding 209 undefined
imports and 0 missing target symbols, with a zero-length `__versions` section.

```text
android14-6.1_kernelsu-e1q-S921USQS6DZF2-kdp.ko
size: 400152
SHA-256: c8293f8375ef736f42bce654b0deafa70cccc62286575153c2a49c14098425c9

ksud-e1q-S921USQS6DZF2-kdp
size: 4779088
SHA-256: 3cbf42ba320785078c65b9a9b14b3d5331674927ade771a2134df0738376631d
```

## Support feed

Entry `e1q-S921USQS6DZF2` in `support/targets-v3.json`:
- models: `SM-S921U`, `SM-S921U1`
- kernel versions: `6.1.145`
