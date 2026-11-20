# memcomp 内存压缩补丁（6.12.69 GKI 重制版）

本目录存放 [memcomp](https://github.com/nakixii/memcomp) 在 **ACK android16-6.12-2026-03（6.12.69）** 源码树上的重制补丁，
供 `fastbuild_6.12.69_gki_memcomp.yml` 与 `local/builder_6.12.69_gki_memcomp.sh` 在编译前应用。

## 来源

| 项目 | 值 |
| --- | --- |
| 上游仓库 | `https://github.com/nakixii/memcomp`（RFC v1，纯内存压缩后备存储，替代 zram/zswap） |
| 内核分支 | `https://github.com/nakixii/linux` 的 `memcomp/android16-6.12`，分支头 `713abbe0490a3`（tree `d37a8c4e747f`） |
| 该分支基线 | ACK `android16-6.12` @ `738f95cebf10`（6.12.23） |
| 本补丁基线 | `cctv18/android_gki_kernel_common` 的 `android16-6.12-2026-03` 分支（6.12.69，commit `58ee67741556`） |

补丁内容等价于上游分支 `738f95ce..713abbe` 的全部 27 个提交（9 个前置回移 + 16 个系列补丁 + 2 个 Android 补丁），
在 6.12.69 上重新落位后的**单个合并补丁**：

- 系列补丁：swap entry 新类型、匿名页内存内压缩、memcg 计费、shmem/tmpfs 页、冷页回写 swap、
  前台批量异步压缩、大页批量 unmap、KUnit 与 selftest、设计文档与管理指南；
- Android 补丁：`gki_defconfig` 开启 `CONFIG_MEMCOMP` 并关闭 zram、shmem 回收路径保留
  `trace_android_vh_shmem_mod_swapped` 与 smaps `swap_shared` 统计；
- 前置回移：mTHP/MTE/zero page 相关的 6.12 回移。**其中 `cpufeature.c`、`mte.c` 的改动 6.12.69 已包含，
  故本补丁不再重复携带**；`contpte.c`、`rmap.c`、`huge_memory.c`、`migrate.c` 等处的批处理改动按 6.12.69 现状重新落位。

## 与上游的差异（重制说明）

6.12.23 → 6.12.69 之间 `mm/` 有较多稳定版回移，上游补丁的少数补丁块无法直接落位，重制时按下述方式处理：

1. `arch/arm64/configs/gki_defconfig`：6.12.69 的 zram 配置项更多（`ZRAM_BACKEND_ZSTD`、`ZRAM_WRITEBACK`、
   `ZRAM_MULTI_COMP`、`ZRAM_ANDROID_IOCTL`），一并移除，改为 `CONFIG_MEMCOMP=y` + `# CONFIG_ZRAM is not set`；
2. `mm/huge_memory.c`：`thp_underused()` 采用上游按 folio 阶数缩放 `max_ptes_none` 的版本，并补回
   `thp_still_young()` / `thp_unqueue_used()`（6.12.69 已有 `pages_identical()`、`folio_test_has_hwpoisoned()`）；
3. `mm/rmap.c`：`folio_referenced_one()` 保留 6.12.69 的 `trace_android_vh_look_around` 厂商钩子，
   同时按上游改为 `folio_pte_batch()` 批量取 `nr` + `clear_flush_young_ptes_notify()` / `lru_gen_look_around(pvmw, nr)`；
4. `mm/shmem.c`：6.12.69 的 `shmem_free_swap()` 已改为 xarray 实现，memcomp token 按“一页”语义改写，
   并把 `free_swap_and_cache_nr()` 换成 `memcomp_invalidate_cache()`；
5. `mm/migrate.c`：`__migrate_folio()` 中 memcomp 迁移 + 回滚逻辑按 6.12.69 结构落位；
6. `mm/vmscan.c`：`shrink_folio_list()` 末尾的 `keep_locked`/`keep` 与 `memcomp_finish_deferred()` 按 6.12.69 结构落位；
7. `mm/userfaultfd.c`：6.12.69 的 `move_swap_pte()` 增加了 `si`/`entry` 两个参数（swap cache 竞态修复），
   memcomp 路径以 `NULL, NULL, entry` 调用，并在函数内改成 `else if (si)` 跳过 `si->swap_map[]` 检查
   （memcomp entry 没有 swap 设备）；`move_pages_ptes()` 的返回值变量在 6.12.69 叫 `ret` 而非 `err`；
8. `arch/arm64/mm/contpte.c`：上游补丁删除的 `pgprot_t prot;` 在 6.12.23 属于 `contpte_ptep_get_lockless()`
   （6.12.69 已改用 `contpte_is_consistent()`），该删除被 patch 按上下文误落到 `contpte_set_ptes()`，
   重制时保留 `contpte_set_ptes()` 的 `prot` 声明。

## 应用方式

```sh
cd common
patch -p1 -F 3 --no-backup-if-mismatch < ../memcomp_patch/memcomp-android16-6.12.patch
```

补丁必须**在其他第三方补丁（susfs、lz4/zstd、KSU 相关）之前**应用，以保证与官方源码上下文一致；
已验证 susfs4oki（`oki-android16-6.12`）可以在本补丁之后正常应用（`mm/memory.c` 的头文件块以 fuzz 3 落位）。

## 配置要求

```
CONFIG_MEMCOMP=y
CONFIG_MEMCOMP_DEFAULT_COMPRESSOR="zstd"
CONFIG_SWAP=y
# CONFIG_ZRAM is not set
# CONFIG_ZSWAP is not set
```

- memcomp 与 **zram / zswap 互斥**，编译出的内核不再有 zram，系统不再有传统 zram swap；
- 压缩算法通过内核 crypto acomp 提供，需内置：`CONFIG_CRYPTO_ZSTD=y`（默认）、`CONFIG_CRYPTO_LZ4=y`、
  `CONFIG_CRYPTO_LZ4HC=y`、`CONFIG_CRYPTO_LZO=y`、`CONFIG_CRYPTO_DEFLATE=y`、`CONFIG_CRYPTO_842=y`；
- 运行时可 `memcomp.compressor=<name>` 覆盖默认算法，`/sys/kernel/mm/memcomp/{compressor,stat}` 查看状态。

## 验证情况

- 补丁已在 6.12.69 源码树上逐块应用并人工解决冲突，`patch -p1 -F 0` 可完整、无 fuzz 地重放（回环验证通过）；
- 已确认 6.12.69 保留了全部 `trace_android_vh_*` 厂商钩子，未被本补丁删除；
- 本补丁**未经本地编译验证**（Windows 环境无 Linux/clang 工具链），首次编译请在 CI 或本地脚本中确认；
  若上游 memcomp 更新，请按同样方式重新生成补丁，而不是直接套用上游分支。
