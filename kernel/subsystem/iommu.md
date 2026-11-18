# IOMMU Subsystem Details

This guide covers cross-architecture invariants, API contracts, and recurring
bug patterns for the core IOMMU subsystem (`drivers/iommu/`,
`drivers/iommu/generic_pt/`, `drivers/iommu/iommufd/`, and
`include/linux/iommu.h`), including domain attachment, DMA mapping, Shared
Virtual Addressing (SVA), I/O Page Faults (IOPF), `generic_pt`, and `iommufd`.
For Intel VT-d, AMD-Vi, and Hyper-V x86 drivers, also load `iommu-x86.md`.

## Domain Attachment, Replacement, and Release Lifecycle

Mismanaging the transition from `@old` to `@new` during domain attachment or
device release leaves hardware translating through freed page tables, drops
active I/O page-fault handlers or MSI reservations on failed replacements, or
corrupts per-domain lists shared across multiple IOMMU groups.

- **Use the core-provided `@old` domain parameter, never cached `dev_iommu_priv` state:**
  `.attach_dev(domain, dev, old)` and `.set_dev_pasid(domain, dev, pasid, old)`
  transition directly between `IOMMU_DOMAIN_PAGING`, `IOMMU_DOMAIN_IDENTITY`,
  `IOMMU_DOMAIN_BLOCKED`, and `IOMMU_DOMAIN_NESTED` without an intermediate
  detach callback (`@old` is non-NULL whenever replacing an active attachment).
  - Driver helpers invoked early during attach or error fallback (for example,
    `device_block_translation()` in `drivers/iommu/intel/iommu.c`) frequently
    clear or overwrite the driver's private `info->domain` / `dev_data->domain`
    before teardown of the previous domain finishes. Always inspect and clean up
    the `@old` pointer passed by the IOMMU core rather than re-reading private
    per-device state.
  - Group-level PASID replacement (`__iommu_set_group_pasid()` in
    `drivers/iommu/iommu.c`) rolls back already-updated group devices by calling
    `old->ops->set_dev_pasid(old, dev, pasid, domain)` if a later device in the
    group fails; `.set_dev_pasid` implementations must never fail when restoring
    a previously valid `@old` domain.

- **Transactional `@old` to `@new` state replacement:**
  Per-device auxiliary state tied to the attached domain — such as IOPF
  participation (`iopf_for_domain_set()` / `iopf_for_domain_replace()`) or
  `iommufd` IOAS reserved IOVA intervals (`iommufd_device_do_replace()` in
  `drivers/iommu/iommufd/device.c`) — must not be destructively removed from
  `@old` before programming `@new` is guaranteed to succeed.
  - If `@old` and `@new` share an underlying container (such as two HWPTs on the
    same `iommufd_ioas`), removing `@old`'s per-device reserved regions (e.g.,
    the x86 `0xfee00000` MSI window) before installing `@new`'s regions — or
    dropping `@old`'s IOPF refcount without rolling back on error — leaves `@old`
    active in hardware with missing IOVA reservations or unbalanced fault queues
    when attach fails. Use atomic replace helpers (`iopf_for_domain_replace()`)
    and restore `(old, new, dev)` on failure.

```c
// WRONG: tears down @old fault state before @new hardware setup can fail
if (old && old->iopf_handler)
	iopf_for_domain_remove(old, dev);
ret = program_hw_attachment(domain, dev);
if (ret)
	return ret; // @old remains attached in HW with IOPF torn down!

// CORRECT: replace transactionally and roll back to @old on failure
ret = iopf_for_domain_replace(domain, old, dev);
if (ret)
	return ret;
ret = program_hw_attachment(domain, dev);
if (ret) {
	iopf_for_domain_replace(old, domain, dev);
	return ret;
}
```

- **Intentional `.release_domain` override for `IOMMU_RESV_DIRECT` (do NOT flag as a bug):**
  In `iommu_deinit_device()` (`drivers/iommu/iommu.c`), even when a driver sets
  `ops->release_domain = &blocked_domain`, the core overrides `release_domain`
  to `ops->identity_domain` whenever `dev->iommu->require_direct` is set (devices
  with `IOMMU_RESV_DIRECT` / RMRR / IVMD unity-map requirements that must keep
  DMA access alive across driver unbind).
  - Consequently, `.release_device(dev)` is NOT guaranteed to observe a
    detached/`NULL` or `blocked_domain` pointer on entry. Drivers must handle a
    still-attached identity domain in `.release_device()` and must never assert
    `WARN_ON(dev_data->domain != NULL)`.

- **Domain-to-group multiplicity (`group->mutex` is per-group, not per-domain):**
  Default DMA domains are 1:1 with `struct iommu_group`, so `group->mutex`
  serializes single-group operations. However, VFIO Type1 (`drivers/vfio/vfio_iommu_type1.c`)
  and `iommufd` attach **multiple `iommu_group`s to a single `iommu_domain`**.
  Any mutable per-domain collection mutated during attach, map, or MSI setup
  (such as `cookie->msi_page_list` or `domain->dev_list`) cannot rely solely on
  `group->mutex` for mutual exclusion across groups.

- **Deferred attachment (`iommu_is_attach_deferred`) before early mappings:**
  In kdump kernels (`iommu_is_attach_deferred()`), hardware continues using the
  crashed kernel's translation tables until the new driver initiates DMA. Any
  path that populates IOVA mappings in a device's default domain prior to normal
  `dma_map_*()` calls — including MSI doorbell preparation in
  `iommu_dma_prepare_msi()` (`drivers/iommu/iommu.c`) — must call
  `iommu_deferred_attach(dev, domain)` **outside** `group->mutex` (since
  `iommu_deferred_attach()` acquires `group->mutex` internally) so the newly
  mapped IOVA exists in the page table hardware actually walks.

**REPORT as bugs:**
- Reading a driver's private cached `dev->domain` pointer instead of the `@old`
  parameter inside `.attach_dev()` or `.set_dev_pasid()` after helper calls may
  have mutated private device state.
- Removing `@old`'s IOPF registration or shared IOAS IOVA reservations before
  fallible `@new` attachment steps without restoring `@old`'s state on failure.
- Asserting `WARN_ON(priv->domain)` inside `.release_device()` on a driver that
  supports `ops->identity_domain`, ignoring `dev->iommu->require_direct`.
- Calling `iommu_deferred_attach()` while already holding `dev->iommu_group->mutex`
  (self-deadlock), or omitting `iommu_deferred_attach()` on paths that install
  required mappings before first DMA.
- Protecting mutable per-`iommu_domain` lists or counters solely with
  `iommu_group->mutex` when the domain type can be attached to multiple groups.

## SVA, IOPF, and `iommufd` Object Publication and Lifetime

Concurrent fault-reporting threads and `iommufd` ioctls look up attach handles,
IOVA areas, and objects locklessly or as soon as locks drop. Publishing
half-initialized handles, sharing list nodes across core and handler queues, or
dereferencing context pointers after dropping object user counts causes
use-after-free, double responses, and NULL pointer dereferences.

- **Complete initialization before XArray / tree publication or lock release:**
  - `iommu_attach_device_pasid()` (`drivers/iommu/iommu.c`) inserts `@handle`
    into `group->pasid_array`, making it immediately visible to lockless
    `iommu_attach_handle_get()` readers in IRQ/fault paths. Callers such as
    `iommu_sva_bind_device()` (`drivers/iommu/iommu-sva.c`) must initialize
    `handle->dev`, refcounts, and driver fields *before* publishing or dropping
    serialization locks, and callers must always pass a fresh
    `struct iommu_attach_handle` rather than reusing one still reachable by
    in-flight fault readers.
  - In `drivers/iommu/iommufd/`, objects inserted into `ictx->objects` or
    `viommu->vdevs` XArrays, and `struct iopt_area` nodes inserted into the
    IOVA interval tree (`drivers/iommu/iommufd/ioas.c`), become visible as soon
    as the protecting lock drops. Note that `iopt_alloc_area_pages()` inserts an
    `iopt_area` with `area->pages == NULL` under `iova_rwsem`, drops
    `iova_rwsem`, and populates `area->pages` later under `domains_rwsem`; any
    walker holding only `iova_rwsem` MUST guard against `!area->pages`.

- **Exclusive list ownership for `struct iopf_group`:**
  In `drivers/iommu/io-pgfault.c`, `iopf_group_alloc()` places every completed
  fault group onto `iopf_param->faults` via `group->pending_node` so teardown
  (`iopf_queue_remove_device()`) can abort unclaimed groups.
  - Any asynchronous `domain->iopf_handler` (such as `iommufd_fault_iopf_handler()`
    in `drivers/iommu/iommufd/eventq.c`) that queues `group` onto its own
    delivery list (`fault->common.deliver`) MUST first remove it from
    `iopf_param->faults` via `iopf_group_dequeue(group)`. Leaving `group` on
    both lists causes device detach (`iopf_queue_remove_device()`) and fault-queue
    teardown (`iommufd_auto_response_faults()`) to both send a hardware page
    response and double-free `group`.

- **Latch parent context before `refcount_dec(&obj->users)` in file `.release`:**
  In `iommufd` file-backed objects (such as `iommufd_eventq_fops_release()` in
  `drivers/iommu/iommufd/eventq.c`), `IOMMU_DESTROY` on another CPU waits only
  for `obj->users` to drop to zero before freeing the object. Once
  `refcount_dec(&eventq->obj.users)` executes, `eventq` may be freed
  immediately; capture `struct iommufd_ctx *ictx = eventq->ictx;` into a local
  variable *before* decrementing `obj.users`.

- **Never flush IOPF workqueues under locks required by fault handlers:**
  `iopf_queue_flush_dev()` calls `flush_workqueue()` on fault workers whose
  handlers (`iommu_sva` `handle_mm_fault()`, `iommufd` fault delivery) sleep and
  acquire `mm` or domain locks. Never call `iopf_queue_flush_dev()` while
  holding spinlocks or global mutexes/rwsems that fault paths or concurrent
  attach/detach paths acquire.

**REPORT as bugs:**
- Enqueuing a `struct iopf_group` onto a custom/async handler list without
  calling `iopf_group_dequeue(group)` first.
- Dereferencing any field of a `struct iommufd_object` subclass (such as
  `obj->ictx`) after `refcount_dec(&obj->users)` without holding a separate
  owning reference.
- Walking `iopt_area` interval trees in `drivers/iommu/iommufd/` under
  `iova_rwsem` alone while assuming `area->pages` is non-NULL.
- Calling `iopf_queue_flush_dev()` under a spinlock or under a lock acquired by
  the domain's `iopf_handler`.

## Generic Page Tables (`generic_pt`), Dirty Tracking, and DMA Attributes

Type-conversion traps between negative `-errno` helpers and unsigned physical
addresses, partial clearing of contiguous/superpage dirty bits, and lost MMIO
attributes silently corrupt DMA targets, break VM live migration, or trigger
platform memory-encryption faults on PCIe BARs.

- **`iova_to_phys()` returns `0` on failure, NEVER a negative `-errno`:**
  `struct iommu_domain_ops.iova_to_phys` returns `phys_addr_t` (`u64` / `u32`),
  where **`0` indicates no translation exists**. Internal `generic_pt` helpers
  (`make_range()`, `pt_walk_range()`, `__do_iova_to_phys()` in
  `drivers/iommu/generic_pt/iommu_pt.h`) return negative `-errno` (`-ENOENT`,
  `-ERANGE`, `-EOVERFLOW`). Returning `ret` directly from `iova_to_phys()`
  implicitly casts `-ERANGE` to `0xffffffffffffffdeULL`, which callers treat as
  a valid translated physical address. Note: mapping physical address `0` is a
  known corner case of the `iova_to_phys` ABI, but negative error codes must
  always map to `0`.

```c
// WRONG: negative -errno implicitly converts to a huge valid phys_addr_t!
ret = make_range(common, &range, iova, 1);
if (ret)
	return ret;

// CORRECT: return 0 whenever range validation or page-table walk fails
ret = make_range(common, &range, iova, 1);
if (ret)
	return 0;
```

- **Dirty-bitmap read-and-clear atomicity across superpages and contiguous PTEs:**
  When `read_and_clear_dirty()` runs without `IOMMU_DIRTY_NO_CLEAR`
  (`drivers/iommu/generic_pt/`, `drivers/iommu/iommufd/io_pagetable.c`,
  `drivers/iommu/iommufd/iova_bitmap.c`), clearing hardware write-dirty state via
  `pt_entry_make_write_clean()` is destructive and irreversible:
  1. A single large leaf or contiguous PTE group (`pt_entry_num_contig_lg2()`)
     tracks dirtiness as **one hardware unit**. Both `iova_bitmap_set()` and
     `iommu_iotlb_gather_add_range()` must cover the **entire** PTE / contiguous
     entry IOVA span — never clamp the cleared PTE to a sub-range requested by
     the caller without either rejecting unaligned sub-PTE requests or recording
     the full span.
  2. `iova_bitmap` pins user bitmap pages lazily as the walk advances
     (`iova_bitmap_get()` in `drivers/iommu/iommufd/iova_bitmap.c`). Any
     `pin_user_pages_fast()` failure (`bitmap->err`) must abort the walk and
     propagate `-EFAULT` to userspace; silently skipping `iova_bitmap_set()`
     after clearing PTE dirty bits causes silent guest memory corruption during
     live migration.
  3. Always reject `length == 0` *before* computing `iova + length - 1` with
     `check_add_overflow()`, as `length - 1` wraps to `SIZE_MAX` when
     `length == 0`.

- **Preserving `IOMMU_MMIO` / `DMA_ATTR_MMIO` across domain copies and `dma-iommu`:**
  - IOMMU page tables do not always encode a distinct retrievable bit that
    distinguishes `IOMMU_MMIO` (peer-to-peer PCIe BARs / dmabufs) from normal
    RAM PFNs once installed. In `drivers/iommu/iommufd/pages.c`, populating a
    new domain must source PFNs and `BATCH_MMIO` flags from the canonical
    `iopt_pages` / dmabuf physical range metadata rather than reading PFNs back
    out of an existing HWPT; dropping `IOMMU_MMIO` maps BARs as cacheable memory
    on ARM or sets the SME/SEV C-bit on x86.
  - In `iommu_dma_map_phys()` (`drivers/iommu/dma-iommu.c`), `DMA_ATTR_MMIO`
    mappings represent device BAR addresses rather than CPU RAM backed by
    `struct page` or bounce buffers; skip `swiotlb` / untrusted bounce-buffer
    checks when `attrs & DMA_ATTR_MMIO` is set.

- **Scatterlist length overflow and DMA pool return checks (`drivers/iommu/dma-iommu.c`):**
  - `struct scatterlist.length` and `.dma_length` are `unsigned int` (32-bit).
    When `iommu_dma_map_sg()` adds IOVA alignment offsets (`iova_off`) or pads
    size up to granule boundaries (`iommu_page_align()`), check
    `check_add_overflow()` / `overflows_type(..., unsigned int)` before storing
    back into `sg` fields or coalescing adjacent segments.
  - `dma_alloc_from_pool()` returns `struct page *` (`NULL` on failure), while
    its `void **cpu_addr` argument is an output parameter only; check the
    returned `page` pointer, not `*cpu_addr`.

**REPORT as bugs:**
- Returning a negative `-errno` `int` directly from an `.iova_to_phys()`
  callback instead of `0`.
- Clearing hardware PTE dirty bits (`pt_entry_make_write_clean()` or driver
  equivalent) while dropping `iova_bitmap` pin errors or recording/flushing only
  part of a superpage or contiguous PTE span.
- Computing `iova + length - 1` before checking `length == 0`.
- Reconstructing mappings for a secondary domain from an existing IOMMU page
  table in a way that strips `IOMMU_MMIO` from MMIO/dmabuf ranges.
- Adding alignment padding or coalesced lengths into 32-bit `sg->length` /
  `sg->dma_length` without overflow checks.

## Quick Checks

- **User-supplied invalidation descriptors on shared hardware queues:** Nested
  `cache_invalidate_user` callbacks (`IOMMU_HWPT_INVALIDATE` in `iommufd`) submit
  guest-crafted descriptors onto a host-shared command/invalidation queue. Every
  reserved bit, opcode/type whitelist, and target Domain ID / ASID / Device ID
  must be validated or overwritten by the host driver before enqueueing so a
  guest cannot stall the host queue or invalidate another tenant's translations.
- **Rate-limit all device- or guest-triggerable IOMMU logs:** Fault handlers,
  PRI/PPR queues, invalidation completions, and error interrupts can be flooded
  by malfunctioning or malicious endpoints; always use `dev_err_ratelimited()` /
  `pr_warn_ratelimited()`, never unratelimited `printk` or `WARN_ON` on
  device-controlled inputs.
- **Multi-IRQ setup unwind loops:** When an IOMMU unit requests multiple IRQs in
  a loop (event, PRI, command-sync, PMU), verify the failure label frees only
  the indices `[0, i)` actually registered — off-by-one loop bounds or calling
  `free_irq()` on unrequested vectors is a recurring init error-path bug.
