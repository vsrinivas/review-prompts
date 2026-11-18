# x86 IOMMU Subsystem Details

This guide covers invariants, hardware protocols, and recurring bug patterns for
x86 IOMMU drivers: Intel VT-d (`drivers/iommu/intel/`), AMD-Vi
(`drivers/iommu/amd/`), Hyper-V paravirtualized IOMMU (`drivers/iommu/hyperv/`),
and early x86 DMA/IOMMU boot setup (`arch/x86/kernel/pci-dma.c`). Load alongside
`iommu.md`.

## Kdump and Pre-Enabled Translation Tables

When booting a kdump crash kernel (`is_kdump_kernel()`, `translation_pre_enabled()`),
IOMMU hardware is actively translating ongoing DMA from the crashed kernel using
tables in old-kernel physical memory. Dereferencing those tables via linear-map
helpers, misindexing scalable context tables, or leaving old Domain IDs
unreserved causes immediate crash-kernel page faults, IOTLB/PASID aliasing, or
SEV-SNP hardware faults.

- **Never call `phys_to_virt()` on old-kernel tables; always `memremap()`:**
  Root tables, Context tables, PASID directories, PASID tables, AMD Device
  Tables (`dev_table`), and event buffers read from hardware registers in a
  kdump kernel reside outside the crash kernel's linear map.
  - Map every level with `memremap(phys, size, MEMREMAP_WB)` and pair every path
    with `memunmap()`.
  - Do NOT call normal runtime accessors such as `get_pasid_table_from_pde()`
    (`drivers/iommu/intel/pasid.h`) while walking old-kernel tables, because
    those helpers internally call `phys_to_virt()` on physical pointers stored
    in old-kernel PASID directory entries (see `copy_pasid_table_dids()` in
    `drivers/iommu/intel/iommu.c`).

- **VT-d Legacy vs. Scalable Mode (`DMA_RTADDR_SMT`) kdump geometry and DIDs:**
  In `copy_translation_tables()` and `copy_context_table()`
  (`drivers/iommu/intel/iommu.c`), table layout and Domain ID (DID) location
  differ fundamentally between legacy mode and scalable mode (`ext == true`):
  1. **Where DIDs live:** In legacy mode, the DID is stored in the Context Entry
     (`context_domain_id(&ce)`). In scalable mode, Context Entries hold only a
     PASID Directory pointer (`context_pasid_table_address()`) and carry NO DID;
     the active DIDs reside inside each present 512-bit PASID Table Entry
     (`pasid_get_domain_id()`, walked via `copy_pasid_table_dids()`).
  2. **Lower (`LCTP`) vs. Upper (`UCTP`) context-table slots:** Legacy mode uses
     1 context table per bus (`256` devfns). Scalable mode splits each bus's Root
     Entry into a Lower Context Table (`re.lo`, `devfn < 0x80`, stored at
     `tbl[bus * 2 + 0]`) and an Upper Context Table (`re.hi`, `devfn >= 0x80`,
     stored at `tbl[bus * 2 + 1]`). Buses populated only by `devfn >= 0x80`
     devices legitimately have `LCTP == 0` and non-zero `UCTP`; slot indexing
     must always key on `(devfn >= 0x80)` rather than compacting into slot `0`.

- **DID reservation (`reserve_domain_id`) semantics and all-or-nothing rollback:**
  - **Intentional `-ENOSPC` handling (do NOT flag as a bug):**
    `reserve_domain_id()` in `drivers/iommu/intel/iommu.c` calls
    `ida_alloc_range(&iommu->domain_ida, did, did, GFP_KERNEL)`. Returning
    `-ENOSPC` means `did` was **already reserved** by an earlier device or PASID
    sharing that domain in the crashed kernel and is treated as success (`0`).
    Any other negative error (such as `-ENOMEM`) is a real failure and MUST be
    propagated.
  - **Abort and clean up completely on any table-copy failure:** If any
    `memremap()`, allocation, or `reserve_domain_id()` call fails midway through
    `copy_context_table()` or `copy_pasid_table_dids()`, `copy_translation_tables()`
    must abort completely: tear down all newly allocated `ctxt_tbls[]`,
    `bitmap_free(iommu->copied_tables)` and set `iommu->copied_tables = NULL`,
    call `ida_destroy(&iommu->domain_ida)`, and clear `DMA_GCMD_TE` so the IOMMU
    falls back cleanly. Note that `init_dmars()` logs and continues on
    `copy_translation_tables()` failure without freeing `struct intel_iommu`, so
    leaving `copied_tables` or partially reserved IDAs behind leaks memory or
    runs with unreserved DIDs that alias stale IOTLB entries.

- **AMD-Vi kdump table reuse, fallback, and `GFP_DMA32` (`drivers/iommu/amd/init.c`):**
  - **Gate hardware programming on `old_dev_tbl_cpy`, not `is_kdump_kernel()`:**
    `copy_device_table()` / `__reuse_device_table()` can reject the crashed
    kernel's DTE table (for example, on non-SNP systems if table sizes differ
    across kernel versions) and leave `pci_seg->old_dev_tbl_cpy == NULL`,
    falling back to a freshly allocated `dev_table`. Callers such as
    `early_enable_iommu()` must check `!iommu->pci_seg->old_dev_tbl_cpy` (not
    `!is_kdump_kernel()`) when deciding whether to disable the IOMMU and call
    `iommu_set_device_table()`. On SEV-SNP hosts, the DTE base register is
    hardware-locked, so `get_dev_table_size()` must match the host kernel's
    sizing or kdump must panic explicitly rather than silently misprogramming.
  - **Never pass `GFP_DMA32` in kdump kernels:** `crashkernel=X,high
    crashkernel=0,low` configurations have zero memory below 4GB. Normal kernels
    use `GFP_DMA32` in `alloc_dev_table()` so a future crash kernel can trust
    `< 4GB` pointers, but `alloc_dev_table()` must omit `GFP_DMA32` when
    `is_kdump_kernel()` is true.
  - **Clear inherited hardware bypass bits using raw MMIO capabilities:** Crash
    kernels reusing old tables inherit live MMIO control bits from the crashed
    kernel (such as `PERF_OPT_EN` in `MMIO_PERF_OPT_OFFSET`). Early per-IOMMU
    init (`init_iommu_one()`) runs *before* the global `amd_iommu_efr` union is
    finalized, so feature presence must be read directly from
    `readq(iommu->mmio_base + MMIO_EXT_FEATURES)` and init must fail if clearing
    the bypass register times out.

**REPORT as bugs:**
- Calling `phys_to_virt()` (directly or via helpers like
  `get_pasid_table_from_pde()`) on physical addresses read from pre-enabled
  IOMMU hardware tables in kdump paths instead of `memremap()`.
- Treating a non-`-ENOSPC` negative return (e.g., `-ENOMEM`) from
  `ida_alloc_range()` as success when reserving old-kernel Domain IDs, or
  continuing `copy_translation_tables()` with partially copied/reserved state.
- Indexing VT-d scalable-mode root context slots without separating lower
  (`devfn < 0x80`) and upper (`devfn >= 0x80`) tables, or reading DIDs from
  scalable-mode Context Entries instead of PASID Table Entries.
- Gating AMD `iommu_set_device_table()` on `!is_kdump_kernel()` instead of
  checking whether old device-table reuse (`old_dev_tbl_cpy`) actually succeeded.
- Allocating buffers with `GFP_DMA32` on paths reachable when
  `is_kdump_kernel()` is true.

## Hardware-Walked Descriptors, Bypass Modes, and Cache Coherency

VT-d Context Entries (128-bit), VT-d PASID Entries (512-bit), and AMD-Vi Device
Table Entries (256-bit) are multi-word descriptors fetched asynchronously by
IOMMU hardware, potentially non-coherently with CPU caches. Bulk-zeroing a live
entry, updating qwords in the wrong order, or leaving hardware bypass registers
enabled allows endpoints to DMA through torn pointers, freed page tables, or
untranslated physical memory.

- **VT-d multi-word teardown handshake (VT-d Spec 6.5.3.3):**
  When tearing down a live Context Entry (`domain_context_clear_one()`,
  `copied_context_tear_down()` in `drivers/iommu/intel/iommu.c`) or Scalable
  PASID Entry (`pasid_clear_entry()` / `pasid_clear_entry_with_fpd()` in
  `drivers/iommu/intel/pasid.h`):
  1. Clear **only** the `Present` bit (`context_clear_present()` /
     `pasid_clear_present()`, setting `FPD` simultaneously if suppressing faults).
  2. Flush the CPU cacheline (`__iommu_flush_cache()`, which calls
     `clflush_cache_range()` when `!ecap_coherent(iommu->ecap)`) so non-coherent
     IOMMU walkers observe `Present == 0` in RAM.
  3. Invalidate the Context Cache / PASID Cache, IOTLB, and Device-TLB, and wait
     for queued invalidation completion so hardware drops cached translations
     and drains in-flight table walks.
  4. Zero the remaining descriptor qwords (`context_clear_entry()`) and call
     `__iommu_flush_cache()` again.

```c
// WRONG: zeroing all qwords at once lets HW fetch P=1 with a zeroed pointer/DID
context_clear_entry(context);
__iommu_flush_cache(iommu, context, sizeof(*context));
iommu->flush.flush_context(iommu, did, sid, DMA_CCMD_MASK_NOBIT,
			   DMA_CCMD_DEVICE_INOSID);

// CORRECT: clear P=0 -> flush cache -> invalidate HW caches -> clear rest -> flush
context_clear_present(context);
__iommu_flush_cache(iommu, context, sizeof(*context));
iommu->flush.flush_context(iommu, did, sid, DMA_CCMD_MASK_NOBIT,
			   DMA_CCMD_DEVICE_INOSID);
iommu->flush.flush_iotlb(iommu, did, 0, 0, DMA_TLB_DSI_FLUSH);
context_clear_entry(context);
__iommu_flush_cache(iommu, context, sizeof(*context));
```

- **AMD-Vi 256-bit DTE atomic updates (`drivers/iommu/amd/iommu.c`):**
  AMD IOMMU hardware fetches 256-bit `struct dev_table_entry` descriptors in
  128-bit atomic granules (`X86_FEATURE_CX16` / `cmpxchg16b` is mandatory). All
  updates to live DTEs (`set_dte_entry()`, `clear_dte_entry()`, IRTE pointer
  updates) must transition through `amd_iommu_update_dte()` followed by
  `iommu_flush_dte()` and `iommu_completion_wait()`. Never `memset()` or write
  individual `u64` words directly into an active `dev_table` slot.

- **Disable hardware bypass registers BEFORE DTE/Context blocking:**
  On AMD IOMMUs with `MMIO_PERF_OPT_OFFSET` (`PERF_OPT_EN`), hardware routes
  eligible integrated GPU traffic around DTE translation completely; clearing
  `DTE[V/TV]` in `blocked_domain_attach_device()` does NOT block DMA while
  `PERF_OPT_EN` remains set.
  - Both `blocked_domain_attach_device()` and `amd_iommu_attach_device()` (when
    attaching any non-identity domain) must disable `PERF_OPT_EN` via
    `set_device_perfopt(dev_data, false)` *before* updating the DTE, and must
    propagate `-EIO` if the MMIO handshake fails so VFIO or DMA ownership claims
    abort rather than running unisolated.

**REPORT as bugs:**
- Calling `memset()` or `context_clear_entry()` on a live VT-d Context Entry or
  PASID Entry before clearing `Present`, flushing non-coherent CPU caches, and
  completing hardware context/PASID/IOTLB invalidations.
- Modifying a live VT-d Root, Context, or PASID entry without calling
  `__iommu_flush_cache()` (or `clflush_cache_range()`).
- Mutating a live AMD `struct dev_table_entry` directly instead of using
  `amd_iommu_update_dte()` + DTE flush + completion wait.
- Attaching an AMD device to `blocked_domain` or a paging domain without first
  disabling active hardware bypass (`set_device_perfopt(dev_data, false)`) and
  checking its return value.

## Queued Invalidation, Command Rings, and Range Encoding

Bugs in circular invalidation-queue wrap-around, loop-shared flush arguments, or
64-bit shift widths cause permanent Queued Invalidation (`QI`) hangs, skipped
Device-TLB invalidations, or 32-bit truncation of descriptor high bits.

- **Batched VT-d IQE recovery and circular ring wrap-around (`drivers/iommu/intel/dmar.c`):**
  `qi_submit_sync()` enqueues `count` invalidation descriptors plus a trailing
  `wait_desc` across `[index, wait_index]` modulo `QI_LENGTH`. When a descriptor
  triggers an Invalidation Queue Error (`DMA_FSTS_IQE`), hardware halts fetching
  with `DMAR_IQH_REG` pointing at the faulting slot `head_idx`:
  1. `head_idx` (`(dmar_readq(iommu->reg + DMAR_IQH_REG) >> shift) % QI_LENGTH`)
     can land on **any** descriptor in `[index, wait_index]`, and `index >
     wait_index` whenever the batch wraps across the end of the ring. Range
     membership MUST handle wrap-around:
     `index <= wait_index ? (head_idx >= index && head_idx <= wait_index) :
     (head_idx >= index || head_idx <= wait_index)`.
  2. Before clearing `DMA_FSTS_IQE`, recovery (`qi_drain_remaining_descs()`)
     must overwrite `[head_idx, wait_index)` with fenced NOPs, reset `wait_desc`
     status to `QI_ACTIVE`, flush non-coherent caches, execute `wmb()` **before**
     `writel(DMA_FSTS_IQE, iommu->reg + DMAR_FSTS_REG)` restarts hardware
     fetching, and wait for `wait_desc` completion before callers mark batch
     slots `QI_FREE`. Returning early while hardware still owns `[head_idx,
     wait_index]` corrupts recycled ring slots.

- **Do not mutate loop-scoped range parameters in `cache_tag_flush_range()`:**
  In `drivers/iommu/intel/cache.c`, `domain->cache_tags` contains mixed tag
  types (`CACHE_TAG_IOTLB`, `CACHE_TAG_DEVTLB`, `CACHE_TAG_NESTING_IOTLB`,
  `CACHE_TAG_NESTING_DEVTLB`). Overwriting loop-wide `addr` or `mask` variables
  inside one `switch (tag->type)` arm (for example, widening
  `CACHE_TAG_NESTING_DEVTLB` to `addr = 0, mask = MAX_AGAW_PFN_WIDTH`) and
  falling through or continuing pollutes all subsequent tags in the list; keep
  per-tag overrides in local variables scoped to that iteration.

- **64-bit shift hazards and VT-d `S=1` full-range Dev-IOTLB encoding:**
  Full-address-space Dev-IOTLB invalidations (`drivers/iommu/intel/iommu.h`,
  `cache.c`) pass `size_order = MAX_AGAW_PFN_WIDTH` (`64 - VTD_PAGE_SHIFT == 52`):
  - **Shift UB and `X86_32` truncation:** `VTD_PAGE_SIZE << size_order`
    evaluates `1UL << 64` (undefined behavior; masks to `1` on x86_64). Worse,
    `drivers/iommu/intel/` builds on 32-bit `X86_32` where `unsigned long` is
    32 bits: `1UL << (VTD_PAGE_SHIFT + size_order - 1)` (`1UL << 63`) wraps to
    `1UL << 31`, and `~mask` zero-extends to `u64`, wiping bits `[63:32]` of
    `desc->qw1`. Always use `BIT_ULL()` / `GENMASK_ULL()` (`u64`) and skip
    sub-64-bit alignment assertions when `size_order == 64 - VTD_PAGE_SHIFT`.
  - **Never clamp `size_order` to `51` to avoid the shift:** In VT-d `S=1`
    encoding (`qi_desc_dev_iotlb_pasid()`), hardware locates the least-significant
    `0` bit `N` in `ADDR[63:12]` and invalidates `[63:N+1]`. `size_order = 52`
    sets `ADDR[62:12] = 1` and `ADDR[63] = 0` (`N = 63`, matching the entire
    64-bit space). Clamping `size_order` to `51` puts the `0` at bit `62`,
    flushing **only half** of the 64-bit address space (`ADDR[63] == 0`) and
    leaving high canonical/64-bit Dev-IOTLB translations stale!

- **1-based `ffs()` vs. 0-based `__ffs()` on `CAP.SAGAW` bitmaps:**
  `ffs(x)` returns a **1-based** bit position (`0` if none, `1` for bit 0),
  whereas `__ffs(x)` returns a **0-based** bit index (undefined when `x == 0`).
  In VT-d `CAP.SAGAW` (`compute_vasz_lg2_ss()` in `drivers/iommu/intel/iommu.c`),
  bit 1 = 39-bit (3-level), bit 2 = 48-bit (4-level), and bit 3 = 57-bit
  (5-level); mixing `ffs()` and `__ffs()` shifts `agaw` by one page-table level
  (9 IOVA bits).

**REPORT as bugs:**
- Checking circular invalidation-queue index ranges with `head >= start &&
  head <= end` without handling ring wrap-around (`start > end`).
- Clearing `DMA_FSTS_IQE` via `writel()` before issuing `wmb()` after updating
  invalidation queue descriptors in RAM, or freeing QI slots before hardware
  retires the batch's `wait_desc`.
- Using `1UL << ...` or `BIT()` instead of `BIT_ULL()` / `GENMASK_ULL()` on
  page-order or address shifts that can reach `>= 32` (or `== 64`) in x86 IOMMU
  descriptors, or clamping VT-d `MAX_AGAW_PFN_WIDTH` (`52`) to `51` on full
  Dev-IOTLB flushes.
- Mutating caller-supplied `addr` / `mask` variables across iterations of a
  multi-tag flush loop.
- Off-by-one bit arithmetic from using 1-based `ffs()` as a 0-based bit index
  (or calling `__ffs(0)` without a non-zero guard).

## PCI Topology, DMA Aliases, and Device Lookups

PCIe-to-PCI bridges, multifunction DMA-alias quirks, and root-bus integrated
endpoints break assumptions that every device has a single Requester ID (RID) or
a non-NULL parent PCI bridge.

- **Use the callback's alias `(bus, devfn)` SID, never `info->bus/devfn`:**
  Callbacks passed to `pci_for_each_dma_alias(pdev, fn, data)` — such as
  `domain_context_clear_one()`, `device_pasid_table_teardown()`, and
  `context_setup_pass_through()` in `drivers/iommu/intel/iommu.c` and `pasid.c`
  — are invoked once per alias `(bus, devfn)`.
  - Context-cache invalidation (`intel_context_flush_no_pasid()`) MUST receive
    `PCI_DEVID(bus, devfn)` from the callback parameters as `@sid`. Passing
    `PCI_DEVID(info->bus, info->devfn)` flushes the endpoint's primary RID
    repeatedly while leaving alias entries cached in hardware pointing at freed
    PASID tables.

```c
// WRONG: flushes endpoint RID instead of the alias (bus, devfn) being torn down
static int device_pasid_table_teardown(struct pci_dev *pdev, u16 alias, void *data)
{
	struct device_domain_info *info = dev_iommu_priv_get(data);
	...
	intel_context_flush_no_pasid(iommu, PCI_DEVID(info->bus, info->devfn));
	return 0;
}

// CORRECT: flush the exact alias SID whose context entry was cleared
static int device_pasid_table_teardown(struct pci_dev *pdev, u16 alias, void *data)
{
	...
	intel_context_flush_no_pasid(iommu, alias);
	return 0;
}
```

- **Roll back already-programmed aliases on mid-walk setup failure:**
  `pci_for_each_dma_alias()` halts on the first non-zero return after earlier
  aliases in the walk already have live Context Entries pointing at the device's
  PASID directory. If `intel_pasid_setup_sm_context()` fails midway during
  `intel_iommu_probe_device()`, the error path MUST call
  `intel_pasid_teardown_sm_context(dev)` *before* `intel_pasid_free_table(dev)`.

- **Root-bus integrated endpoints have `pci_upstream_bridge(pdev) == NULL`:**
  AMD APU integrated GPUs, FCH integrated endpoints, and Intel Root-Complex
  Integrated Endpoints reside directly on a root bus (`pci_is_root_bus(pdev->bus)`)
  with no upstream bridge, so `pci_upstream_bridge(pdev)` returns `NULL` (see
  `quirks_force_identity_mapping()` in `drivers/iommu/amd/iommu.c`). Always
  NULL-check `pci_upstream_bridge()` before dereferencing bridge fields.

- **`pci_get_domain_bus_and_slot()` reference leaks and PCI BDF formatting:**
  - `pci_get_domain_bus_and_slot()` increments `pdev`'s refcount; every normal
    and error return path (including `-EOPNOTSUPP` / NULL-handler bailouts in
    fault/PPR paths like `iommu_call_iopf_notifier()` in `drivers/iommu/amd/ppr.c`
    and `amd_iommu_report_ppr_err()` in `drivers/iommu/amd/iommu.c`) must call
    `pci_dev_put(pdev)`.
  - Never print `pdev->dev.id` when logging a PCI device's Requester ID (`dev.id`
    is the generic `struct device` platform instance ID, typically `0`); use
    `pci_dev_id(pdev)`, `dev_data->devid`, or `pci_name(pdev)`.

**REPORT as bugs:**
- Passing `PCI_DEVID(info->bus, info->devfn)` instead of the current alias SID
  (`alias` or `PCI_DEVID(bus, devfn)`) inside a `pci_for_each_dma_alias()`
  callback.
- Freeing a PASID table on `intel_pasid_setup_sm_context()` failure without
  tearing down context entries already installed on earlier DMA aliases.
- Dereferencing `pci_upstream_bridge(pdev)` without a `NULL` check.
- Returning after `pci_get_domain_bus_and_slot()` without calling
  `pci_dev_put()` on every exit path, or logging `pdev->dev.id` as a PCI BDF.

## Multi-Stage Boot, Syscore Resume, IRQ Remapping, and Confidential Computing

x86 IOMMU initialization spans early boot-param parsing, pre-PCI-enumeration ACS
enabling, post-enumeration DMA init, decoupled Interrupt Remapping, IRQs-off
`syscore` suspend/resume, and SEV-SNP/SME memory encryption. Skew across these
phases causes peer-to-peer ACS isolation holes, broken IOAPIC/MSI remapping,
sleeping-with-IRQs-off panics, or host Confidential Computing state corruption.

- **Keep early `pci_request_acs()` and late IOMMU enablement conditions identical:**
  `pci_request_acs()` must execute *before* PCI bus enumeration configures
  bridge ACS registers, whereas `intel_iommu_init()` / `amd_iommu_init()` run
  *after* PCI enumeration.
  - Every disable or opt-out condition (`no_iommu`, `iommu_detected`,
    `intel_iommu=off`, `dmar_policy` / `DMAR_REMAP_OPT_OUT` / `DMAR_FW_OFF`,
    `tboot_force_iommu()`, `dmar_platform_optin()`, `amd_iommu=off`, and Xen PV
    skipping `amd_iommu_detect()` without advancing `IOMMU_START_STATE`) must be
    honored consistently in both early detection and late init (`drivers/iommu/intel/`,
    `drivers/iommu/amd/init.c`). Enabling the IOMMU in late init when early
    detect skipped `pci_request_acs()` leaves PCIe downstream ports without ACS
    isolation.
  - Any x86 IOMMU Kconfig target calling `pci_request_acs()` or PCI APIs (such
    as `HYPERV_PVIOMMU` in `drivers/iommu/hyperv/Kconfig`) must declare
    `depends on PCI`.

- **Interrupt Remapping is decoupled from DMA Remapping:**
  Both VT-d (`DMAR_REMAP_OPT_OUT`, `disable_igfx_iommu`) and AMD-Vi
  (`amd_iommu_pgtable == PD_MODE_NONE`) support active Interrupt Remapping when
  DMA translation is disabled or fails per-device probe.
  - DMA probe error paths (`amd_iommu_probe_device()` in
    `drivers/iommu/amd/iommu.c`) must NEVER clear `pci_seg->rlookup_table[devid]`,
    which is populated during ACPI IVRS init and required by Interrupt Remapping
    for that device.
  - **Load-bearing `amd_iommu_irq_remap` check (do NOT flag as redundant):** In
    `drivers/iommu/amd/init.c`, `init_iommu_one_late()` creates and registers
    `iommu->ir_domain` *before* `check_ioapic_information()` runs later in
    `early_amd_iommu_init()`. When `check_ioapic_information()` fails, it clears
    `amd_iommu_irq_remap = false` without destroying the registered `ir_domain`;
    therefore, `if (!amd_iommu_irq_remap) return 0;` in `irq_remapping_select()`
    is essential to avoid matching an orphaned domain and MUST NOT be removed.

- **`syscore_ops` suspend/resume runs with IRQs disabled (`iommu_resume`, `amd_iommu_resume`):**
  `syscore_ops.suspend` and `.resume` run on the boot CPU with **local IRQs
  disabled and non-boot CPUs offline**.
  - Helpers shared between boot init and `syscore` resume (such as
    `enable_iommus_vapic()`, `early_enable_iommu()`, and `iommu_disable()` in
    `drivers/iommu/amd/init.c`) MUST NOT allocate memory (`GFP_KERNEL` sleeps
    with IRQs disabled, and re-allocating on resume leaks and overwrites
    boot-allocated `cmd_buf`, `ppr_log`, or `ga_log` pointers).
  - Suspend helpers (`iommu_disable()`) that clear hardware control bits must
    preserve software configuration state (such as `iommu->perfopt_enabled`) so
    resume restores pre-suspend hardware settings, while teardown helpers
    (`free_ga_log()`) must set freed struct pointers to `NULL`.

- **AMD SEV-SNP and SME invariants (`drivers/iommu/amd/`):**
  - `iommu_snp_enable()` (`drivers/iommu/amd/init.c`) sets
    `CC_ATTR_HOST_SEV_SNP`; any failure jump to `disable_snp:` must clear
    **both** `amd_iommu_snp_en = false` and
    `cc_platform_clear(CC_ATTR_HOST_SEV_SNP)`.
  - Unless hardware advertises `amd_iommu_snp_mode0_sup`
    (`FEATURE_SNP_PAGE_MODE0_SUP`), SEV-SNP forbids `DTE[Mode] = 0`, requiring
    the driver to reject `IOMMU_DOMAIN_IDENTITY`, host SVA, and v2 DMA page
    tables.
  - Under SME (`CC_ATTR_MEM_ENCRYPT`), page-table and GCR3 physical addresses
    installed into hardware structures require `__sme_set()`, whereas PCIe BAR
    mappings (`IOMMU_MMIO`) must NOT set the encryption C-bit.

**REPORT as bugs:**
- Allocating memory with `GFP_KERNEL` or re-allocating ring buffers inside
  functions reachable from `syscore_ops.suspend` or `syscore_ops.resume`.
- Clearing `pci_seg->rlookup_table[devid]` or IRQ-remapping lookup tables from a
  DMA-only device probe error path.
- Adding or modifying an x86 IOMMU boot disable/opt-out flag without keeping
  early `pci_request_acs()` detection and late IOMMU initialization in sync.
- Error paths after setting `CC_ATTR_HOST_SEV_SNP` that clear
  `amd_iommu_snp_en` without calling `cc_platform_clear(CC_ATTR_HOST_SEV_SNP)`.
- Allowing `DTE[Mode] = 0` (identity domains or unguarded v2/SVA host modes) on
  SEV-SNP hosts when `!amd_iommu_snp_mode0_sup`.

## PRI/PPR Fault Protocol, Signed IDs, and Firmware Tables

Protocol mismatches in PCIe Page Request Interface (PRI) completions, signed ID
allocator checks, or untyped ACPI subtable walks cause hung endpoint PRI queues,
leaked error domain IDs programmed into hardware, and false unity-map ranges
under QEMU/KVM.

- **AMD-Vi PPR completion rules and tag bitfields (`drivers/iommu/amd/iommu.c`, `ppr.c`):**
  - **Completing `EVENT_TYPE_INV_PPR_REQ`:** When an invalid PPR event is logged
    in the Event Log with `EVENT_FLAG_PPR_RX == 0` (hardware did not auto-send a
    response), software MUST issue a `CMD_COMPLETE_PPR` via
    `amd_iommu_report_ppr_err()` so the endpoint's PRI request credit is
    returned.
  - **`GN` bit in `COMPLETE_PPR_REQUEST`:** The `GN` bit indicates whether the
    request used Guest/v2 (`GN = 1`) or Host/v1 (`GN = 0`) page tables — derive
    it from `pdom_is_v2_pgtbl_mode(dev_data->domain)` (or
    `flags & EVENT_FLAG_PPR_GN` when no domain is available), NEVER from
    `dev_data->pri_tlp` (an unrelated PCIe TLP prefix capability bit).
  - **9-bit `PPR_TAG` vs. bit-41 `Last` flag:** `PPR_TAG(x)` (`PRGIndex`) is
    9 bits (`0x1ff` at bits `[40:32]`), while bit 41 (`0x200`,
    `PPR_TAG_LAST_PAGE`) is the `L` last-page flag. Masking `PPR_TAG` with
    `0x3ff` corrupts `grpid` on every last-page fault.

- **Store signed ID allocator results in `int` before assigning to `u16`/`u32`:**
  `amd_iommu_pdom_id_alloc()` (`drivers/iommu/amd/iommu.c`, `nested.c`) and
  `ida_alloc_range()` (`drivers/iommu/intel/`) return signed `int` (`< 0` on
  exhaustion or `-ENOMEM`; `0` is also invalid for `hdom_id`). Assigning the
  return value directly to an unsigned struct member such as `u32
  gdom_info->hdom_id` turns `-ENOSPC` into `0xffffffe4U`, bypassing `< 0` /
  `<= 0` error checks and programming a bogus Domain ID into hardware.

```c
// WRONG: gdom_info->hdom_id is u32, so <= 0 never catches -ENOSPC!
gdom_info->hdom_id = amd_iommu_pdom_id_alloc();
if (gdom_info->hdom_id <= 0)
	return -ENOSPC;

// CORRECT: check signed int return before storing into unsigned field
int hdom_id = amd_iommu_pdom_id_alloc();
if (hdom_id <= 0)
	return hdom_id < 0 ? hdom_id : -ENOSPC;
gdom_info->hdom_id = hdom_id;
```

- **Validate ACPI IVRS / DMAR subtable `type` BEFORE reading `flags`:**
  Top-level ACPI IVRS callbacks (`init_memory_definitions()`, `init_iommu_all()`
  in `drivers/iommu/amd/init.c`) receive every subtable header in the table —
  both hardware definitions (`IVHD` types `0x10`, `0x11`, `0x40`) and memory
  definitions (`IVMD` types `0x20`, `0x21`, `0x22`). Flag bits overlap across
  subtable types (`IVHD` `HtTunEn` at bit `0x01`, set unconditionally by QEMU,
  has the exact same bit value as `IVMD_FLAG_UNITY_MAP`). Always check
  `m->type` (e.g., `ivrs_block_type_ivmd(m->type)`) *before* testing `m->flags`
  or casting to a subtable-specific struct. Similarly, validate ACPI DMAR
  subtable lengths and non-NULL device scopes before dereferencing flexible
  arrays (`drivers/iommu/intel/dmar.c`).

**REPORT as bugs:**
- Assigning the return value of `amd_iommu_pdom_id_alloc()`, `ida_alloc*()`, or
  similar signed ID allocators directly into an unsigned (`u16` / `u32`) variable
  before checking `< 0`.
- Inspecting `ivmd_header->flags` or `dmar_header` type-specific fields in an
  ACPI table walk before verifying the subtable `type` discriminant.
- Setting the `GN` bit of an AMD `COMPLETE_PPR_REQUEST` command from
  `dev_data->pri_tlp` instead of v2 page-table mode / `EVENT_FLAG_PPR_GN`, or
  conflating `PPR_TAG_LAST_PAGE` (`0x200`) with the 9-bit `PPR_TAG` (`0x1ff`).
- Dropping `EVENT_TYPE_INV_PPR_REQ` events with `EVENT_FLAG_PPR_RX == 0` without
  issuing `CMD_COMPLETE_PPR`.

## Quick Checks

- **`X86_32` (`i386`) compilation of `drivers/iommu/intel/`:** Intel VT-d
  compiles on 32-bit x86 (`depends on X86 && PCI`). Any bitmask or shift on
  64-bit VT-d registers (`CAP`, `ECAP`, `RTADDR`), page-table entries, or QI
  descriptors (`qw0`, `qw1`) touching bits `>= 32` MUST use `1ULL`, `BIT_ULL()`,
  or `GENMASK_ULL()` — never `1UL` or `BIT()`.
- **Multi-IOMMU heterogeneous capabilities:** A single x86 host frequently has
  multiple DRHD / IOMMU units with different `CAP.SAGAW` (39/48/57-bit),
  coherency (`ecap_coherent()`), or scalable/devtlb capabilities (for example,
  Intel integrated GPU dedicated DRHDs often lack features present on general
  PCIe DRHDs). Never assume global uniform capabilities when attaching or
  allocating nested/second-stage page tables per IOMMU unit.
- **BIOS IVHD EFR vs. hardware `MMIO_EXT_FEATURES`:** Firmware IVHD subtables on
  some AMD systems omit Extended Feature bits advertised in `MMIO_EXT_FEATURES`
  (or vice versa); check whether a feature requires firmware opt-in (`ivhd->efr_attr`)
  vs. hardware capability (`MMIO_EXT_FEATURES`) and ensure early callers do not
  read `amd_iommu_efr` before all units have been probed.
