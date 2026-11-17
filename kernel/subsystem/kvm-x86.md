# x86 KVM Subsystem Details

This guide covers invariants, API contracts, and recurring bug patterns for
`arch/x86/kvm/` across core x86 virtualization, x86 MMU (Shadow and TDP), VMX,
SVM, nested virtualization, paravirtualized interfaces (kvmclock, Xen,
Hyper-V), and Confidential Computing (TDX, SEV/SEV-ES/SEV-SNP). Load alongside
`kvm.md`.

## Quick Checks
- Shared Memory & Double-Fetch: Are guest-shared buffers (GHCB, eVMCS, PV structs) or kvm_run fields read via READ_ONCE() into locals before validation? Are variable-length ioctl structs re-validated after memdup_user()? Are indices validated against actual allocation/bitmap sizes rather than spec maximums?

- Error/Abort Unwinding: On every error path in nested VM-Enter, RSM, vCPU creation, and IRQ routing, is every mutated L1/host field (vmcs01, vmcb01, hflags), pinned/mapped page (kvm_vcpu_map), queued TLB flush, and global table entry (pid_table, producer) properly unwound?

- Teardown & Module Reload: Are workqueues/timers/IOMMU notifiers that reference vCPUs stopped in vm_pre_destroy (before vCPUs are freed)? Are per-vCPU mappings unmapped in vcpu_free? Are kvm.ko globals reset on vendor module reload?
- pCPU Migration & Cross-vCPU Access: Does TLB/VPID/VMCS/ASID logic remain valid if the vCPU migrates pCPUs or if a pCPU hotplugs? Does vcpu_load() avoid guest-visible side effects when !vcpu->wants_to_run? Are lazily-allocated cross-vCPU structs published with smp_store_release() / smp_load_acquire()?

- MMU & Confidential Memory: Do lockless SPTE updates use cmpxchg64 (never atomic bit-ops that corrupt FROZEN_SPTE)? Are staleness checks performed after make_mmu_pages_available()? Are SMM's dual as_id and unaligned memslot boundaries handled? Are firmware/RMP buffers full 4K pages, and are guest_memfd HPAs invalidated on PUNCH_HOLE/memslot flush?

- x86 & Safety Rules: Are GPRs masked for !is_64_bit_mode() (not !is_long_mode())? Is wrmsrq_safe() used for untrusted values? Are BUG_ON() and guest/user-triggerable WARN_ON*() avoided? Are timer/clock deltas protected against overflow/underflow?

- **Inline assembly for VMX instructions requires `"memory"` clobber:**
  `VMREAD`, `VMWRITE`, `VMCLEAR`, and `VMPTRLD` inline `asm` / `asm goto`
  blocks (`arch/x86/kvm/vmx/vmx_ops.h`) access memory-backed VMCS state
  implicitly; omitting the `"memory"` clobber lets the compiler reorder loads
  and stores to `vmcs12` or shadow VMCS memory across `VMPTRLD`/`VMCLEAR`.
- **In-kernel retry loops (`EMULATION_RESTART`, guest PTE `cmpxchg` walk):**
  Loops that stay inside the kernel on guest-provoked retries (such as REP
  string `EMULATION_RESTART` in `arch/x86/kvm/x86.c` or `retry_walk` on guest
  PTE A/D `cmpxchg` failure in `arch/x86/kvm/mmu/paging_tmpl.h`) require
  `cond_resched()`. **Do NOT suggest** checking `kvm_vcpu_has_events()` or
  exiting to the guest on `EMULATION_RESTART`: `pio_in_emulated()` caches
  read-ahead I/O data across restarts, and aborting mid-restart corrupts guest
  I/O state.
  
- **Disappearing interrupts between `kvm_cpu_has_interrupt()` and
  `kvm_cpu_get_interrupt()` are valid with PIC or Xen:** Local APIC interrupts
  cannot vanish asynchronously, so `WARN_ON_ONCE(irq == -1)` is valid only when
  `!pic_in_kernel(kvm)` and Xen evtchn upcalls are inactive. With an in-kernel
  8259A PIC (line de-asserted or stolen by another vCPU) or `CONFIG_KVM_XEN`
  (guest cleared `evtchn_upcall_pending` in shared memory), `get_interrupt`
  legitimately returns `-1` after `has_interrupt` returned true — do not flag
  this race as a bug, and flag any new unconditional `WARN_ON` that assumes
  monotonicity.
  
- **PMU counter reprogramming retries:** In `arch/x86/kvm/pmu.c`, re-requesting
  `reprogram_pmi` when `perf_event_create_kernel_counter()` fails is valid only
  for transient errors (`-EBUSY`, `-ENOMEM`); retrying permanent config
  rejections (`-EINVAL`, `-EOPNOTSUPP`) causes repeated reprogramming overhead.
- **Snapshotting per-vCPU TSC state before enabling preemption/IRQs:** Any
  per-vCPU snapshot read by `vcpu_load()` (such as `vcpu->arch.last_guest_tsc`
  in `arch/x86/kvm/x86.c`) must be written *before* `local_irq_restore()` /
  `preempt_enable()`, otherwise preemption between `rdtsc` and the store races
  `kvm_arch_vcpu_load()`'s TSC catchup logic.
  
- **`KVM_REQ_MASTERCLOCK_UPDATE` vs `KVM_REQ_CLOCK_UPDATE`:** When
  `use_master_clock` is active, `KVM_REQ_MASTERCLOCK_UPDATE` re-snapshots
  `master_kernel_ns` / `master_cycle_now` and introduces a small rate-conversion
  discontinuity in guest kvmclock (`arch/x86/kvm/x86.c`, `arch/x86/kvm/xen.c`).
  Refreshing per-vCPU/shared-info pvclock structures without a host clock epoch
  change should use `KVM_REQ_CLOCK_UPDATE`, not `KVM_REQ_MASTERCLOCK_UPDATE`.

## Guest Entry, Nested Transitions, and Request Lifecycle

Mutating per-vCPU tracking state before hardware entry is guaranteed to
succeed, or clearing pending request/control bits unconditionally after an
aborted entry, causes missed TLB flushes, leaked pinned guest pages, corrupted
host/guest MSRs, NULL MMU dereferences, and re-entry into dead VMs.

- **Failed hardware entry (`VMEXIT_INVALID` / `failed_vmentry` / early bailouts):**
  Hardware can reject `VMRUN` / `VMLAUNCH` / `VMRESUME` before flushing TLBs or
  loading guest state, and vendor `*_vcpu_run()` callbacks can bail out early
  before entering the guest.
  - On SVM (`svm_vcpu_run()` in `arch/x86/kvm/svm/svm.c`), post-entry updates
    that assume the guest actually ran — clearing `vmcb->control.tlb_ctl`,
    clearing `erap_ctl`, marking VMCB control fields clean, or syncing hardware
    MSRs such as `MSR_AMD64_PERF_CNTR_GLOBAL_CTL` back into software state —
    must be gated on `!svm_is_vmrun_failure(svm)`.
  - On VMX/TDX (`arch/x86/kvm/vmx/vmx.c`, `arch/x86/kvm/vmx/tdx.c`), any early
    return from `*_vcpu_run()` before hardware sets an exit reason must stuff
    `exit_reason.full = EXIT_REASON_UNDEFINED` so `*_handle_exit()` never
    dispatches a stale `exit_reason` left over from the previous exit.
  - Treat `exit_reason` as `union vmx_exit_reason`, never as a raw 16-bit
    number or `-1u`. Assigning `-1u` sets upper modifier bits such as
    `failed_vmentry` and `bus_lock_detected`; truncating to `u16` strips them.

- **Nested VM-Enter, RSM, and `!from_vmentry` unwinds:**
  Entering L2 modifies multiple coupled subsystems (mapped `vmcs12`/`vmcb12`
  pages, `vcpu->arch.mmu` switched to `nested_mmu`, `last_vpid` / ASID state,
  MSR/IO bitmaps). Open-coded failure paths that bypass full nested VM-Exit
  frequently miss one of these:
  - Every failure path after mapping nested guest pages must release them
    (`nested_put_vmcs12_pages()` or `CLASS(kvm_vcpu_map_local)` in
    `arch/x86/kvm/vmx/nested.c`).
  - If `last_vpid` or ASID tracking was updated prior to a failed nested
    entry, pending local TLB flush requests (`kvm_service_local_tlb_flush_requests()`)
    must still be serviced before returning to L1.
  - Transitions into L2 that do not originate from `VMLAUNCH`/`VMRESUME`
    (`!from_vmentry`, such as `RSM` out of SMM or `KVM_SET_NESTED_STATE`) do
    not take a synthesized nested VM-Exit on failure; if loading L2 state
    fails after `nested_ept_init_mmu_context()`, the error path must explicitly
    reset `vcpu->arch.mmu` back to `root_mmu` (`nested_ept_uninit_mmu_context()`).

- **Destructive `kvm_check_request()` vs non-destructive `kvm_test_request()`:**
  `kvm_check_request()` atomically tests **and clears** the request bit in
  `vcpu->requests`.
  - If the handler for a request (e.g., `KVM_REQ_GET_NESTED_STATE_PAGES` in
    `arch/x86/kvm/x86.c`) can fail and exit to userspace, it must re-pend the
    request via `kvm_make_request()` on failure so a subsequent `KVM_RUN`
    cannot enter the guest with stale/unpinned state.
  - Never wrap `kvm_check_request()` inside an `#ifdef` unless raising that
    request is compiled out under the exact same `#ifdef`; otherwise disabled
    builds leave the request bit stuck or clear it at the wrong boundary.
  - Terminal sticky flags such as `KVM_REQ_VM_DEAD` must never be consumed with
    `kvm_check_request()`; always inspect them with `kvm_test_request()` so
    subsequent ioctls cannot re-enter a bugged VM.
  - Any helper that queues an injectable exception, interrupt, or nested
    exception VM-Exit (`kvm_queue_exception*()`, `kvm_queue_exception_vmexit()`)
    must raise `KVM_REQ_EVENT` via `kvm_make_request(KVM_REQ_EVENT, vcpu)` so
    `vcpu_enter_guest()` processes the event before entering the guest.

```c
// WRONG: clears request bit permanently even when fallible work fails
if (kvm_check_request(KVM_REQ_GET_NESTED_STATE_PAGES, vcpu)) {
	if (unlikely(!kvm_x86_call(get_nested_state_pages)(vcpu)))
		return 0; // next KVM_RUN skips get_nested_state_pages()!
}

// CORRECT: re-pend the request if servicing it did not complete
if (kvm_check_request(KVM_REQ_GET_NESTED_STATE_PAGES, vcpu)) {
	if (unlikely(!kvm_x86_call(get_nested_state_pages)(vcpu))) {
		kvm_make_request(KVM_REQ_GET_NESTED_STATE_PAGES, vcpu);
		return 0;
	}
}
```

**REPORT as bugs:**
- Clearing `vmcb->control.tlb_ctl`, marking VMCB clean bits, or syncing guest
  MSRs from hardware after `VMRUN` without checking `!svm_is_vmrun_failure()`.
- Early returns in `vmx_vcpu_run()` or `tdx_vcpu_run()` before hardware entry
  that leave `exit_reason` uninitialized or stale instead of setting
  `EXIT_REASON_UNDEFINED`.
- Consuming `KVM_REQ_VM_DEAD` with `kvm_check_request()` instead of
  `kvm_test_request()`.
- Consuming a `KVM_REQ_*` with `kvm_check_request()` before fallible work
  without re-pending the request on failure (unless VM termination follows).
- Queueing a guest exception or nested exception VM-Exit without calling
  `kvm_make_request(KVM_REQ_EVENT, vcpu)`.

## Multi-Context Mode Matrix (SMM, Nested, and CoCo)

Every x86 vCPU executes across orthogonal mode dimensions: normal address space
(`as_id = 0`) vs System Management Mode (`as_id = 1`), L1 (`vmcs01`/`vmcb01`)
vs L2 (`vmcs02`/`vmcb02`), Shadow MMU vs TDP MMU, standard VMX/SVM vs
TDX/SEV-SNP, and Intel vs AMD. Assuming only the default `(AS 0, L1, standard
VM)` slice causes host MSR passthrough leaks, ASID collisions, write-tracking
bypasses, and memory corruption across incompatible vCPU structs.

- **SMM address space (`as_id`) vs shared shadow pages:**
  x86 KVM maintains two memslot address spaces (`0` normal, `1` SMM), each with
  its own `lpage_info` (`disallow_lpage`) and `gfn_write_track` arrays, **but
  shadow pages (`struct kvm_mmu_page`) are shared across both address spaces**.
  - Per-vCPU memslot lookups (such as MMIO caching or GFN translation on a
    vCPU) must use `kvm_vcpu_memslots(vcpu)`, never `kvm_memslots(kvm)` (which
    hardcodes address space `0` and ignores `is_smm(vcpu)`).
  - Any decision about whether a GFN is write-tracked or disallows huge/unsync
    shadow pages (see `kvm_gfn_is_write_tracked()` in
    `arch/x86/kvm/mmu/page_track.c` and `mmu_try_to_unsync_pages()` in
    `arch/x86/kvm/mmu/mmu.c`) must inspect **all** address spaces
    (`slot->as_id ^ 1`), because a shadow page created in AS 0 can be reused or
    aliased in AS 1.

- **L1 vs L2 (`vmcb01`/`vmcb02`, `vmcs01`/`vmcs02`) state sharing:**
  - **MSR and I/O intercept updates while L2 is active:** Dynamic intercept
    changes triggered while `is_guest_mode(vcpu)` is true (for example,
    AVIC/APICv inhibition in `arch/x86/kvm/svm/avic.c` or runtime eVMCS clean
    field updates in `arch/x86/kvm/vmx/nested.c`) must still update L1's
    `vmcb01`/`vmcs01` intercept bitmaps or force a recalculation. Skipping L1's
    bitmap because `is_guest_mode(vcpu)` is currently true leaves host MSRs
    (such as host x2APIC or TPR MSRs) exposed for direct passthrough once L2
    exits to L1.
  - **Active MSRPM selection on SVM:** `is_guest_mode(vcpu)` does NOT guarantee
    `svm->nested.msrpm` is active; when L1 does not intercept MSRs, `vmcb02`
    uses `svm->msrpm`. Check `svm->vmcb->control.msrpm_base_pa` rather than
    branching on `is_guest_mode(vcpu)` (see `msr_write_intercepted()` in
    `arch/x86/kvm/svm/svm.c`).
  - **Shared ASID across VMCBs and CPU hotplug:** In SVM, `svm->asid` is shared
    between `vmcb01` and `vmcb02`, while `vmcb->cpu` is tracked per-VMCB. Any
    pCPU switch or CPU online event (`svm_hardware_enable()`) must invalidate
    ASID state across **both** VMCBs and monotonically increment
    `sd->asid_generation` (never reset it to `1`, which collides with sleeping
    pinned vCPUs).
  - **MMU role resets (`INIT` / `RESET`):** MMU page roles encode `guest_mode`
    and `smm` in addition to `CR0.PG`. Emulating `INIT` (`kvm_vcpu_reset()` in
    `arch/x86/kvm/x86.c`) must unconditionally call `kvm_mmu_reset_context()`
    even if `CR0.PG` was already `0`.

- **VMX vs TDX and Intel vs AMD isolation:**
  - Common Intel VT code (`arch/x86/kvm/vmx/main.c`,
    `arch/x86/kvm/vmx/posted_intr.c`) runs for both VMX (`struct vcpu_vmx`) and
    TDX (`struct vcpu_tdx`). Calling `to_vmx(vcpu)` on a TDX vCPU reinterprets
    `struct vcpu_tdx` as `struct vcpu_vmx` and corrupts memory (`to_vmx` is
    poisoned in shared VT files; dispatch via `is_td_vcpu()` or `vt_x86_ops`).
  - Per-VM features such as Page Modification Logging must check per-VM state
    (`kvm->arch.cpu_dirty_log_size`), not global module parameters
    (`enable_pml`), because TDX VMs coexist with VMX VMs on the same host.
  - Exposing a new `KVM_CAP_*` exit/feature in `arch/x86/kvm/x86.c` or MSRs in
    common `arch/x86/kvm/pmu.c` requires checking whether TDX/SEV-SNP VMs
    support the VMCS/VMCB controls and whether PMU MSRs are Intel- vs
    AMD-specific (`pmu->version == 2` is true on AMD PerfMonV2 as well as Intel
    architectural PMU v2+).

**REPORT as bugs:**
- Calling `kvm_memslots(kvm)` instead of `kvm_vcpu_memslots(vcpu)` in vCPU
  execution, emulation, or per-vCPU MMIO/fault paths where SMM (`as_id = 1`)
  can be active.
- Calling `to_vmx()` on a `struct kvm_vcpu *` without proving `!is_td_vcpu()`
  in code paths reachable by TDX VMs.
- Skipping `vmcb01`/`vmcs01` intercept updates solely because
  `is_guest_mode(vcpu)` is currently true, leaving L1 intercepts stale after
  nested VM-Exit.
- Placing vendor-specific MSRs (such as `MSR_CORE_PERF_GLOBAL_*`) in common
  `arch/x86/kvm/pmu.c` gated only on `pmu->version`.

## x86 MMU Roots, SPTEs, and Rmap Concurrency

x86 MMU page-fault and aging paths combine `mmu_lock` read/write modes,
lockless SPTE atomics, custom rmap bit-spinlocks, and root replacement. Ordering
mistakes here cause slab use-after-free, NULL dereferences on dummy roots,
corrupted `FROZEN_SPTE` synchronization, `PREEMPT_RT` deadlocks/races, and
infinite `RET_PF_RETRY` loops.

- **Ordering `make_mmu_pages_available()` before `is_page_fault_stale()`:**
  `make_mmu_pages_available()` in `arch/x86/kvm/mmu/mmu.c` zaps obsolete pages
  and, under shadow-page pressure, can zap the current vCPU's active root
  (setting `root_sp->role.invalid = 1`). Consequently, `direct_page_fault()`
  and `FNAME(page_fault)()` in `arch/x86/kvm/mmu/paging_tmpl.h` MUST call
  `is_page_fault_stale()` **after** `make_mmu_pages_available()`. Checking
  staleness only before `make_mmu_pages_available()` proceeds to allocate child
  shadow pages under a newly invalidated root (inheriting `role.invalid = 1`
  onto `active_mmu_pages`, see `kvm.md` "Shadow Page `role.invalid`").

```c
// WRONG: make_mmu_pages_available() can invalidate the root after the check
if (is_page_fault_stale(vcpu, fault))
	goto out_unlock;
r = make_mmu_pages_available(vcpu);
if (r)
	goto out_unlock;

// CORRECT: check root/fault staleness after make_mmu_pages_available()
r = make_mmu_pages_available(vcpu);
if (r)
	goto out_unlock;
if (is_page_fault_stale(vcpu, fault))
	goto out_unlock;
```

- **Dummy roots (`kvm_mmu_is_dummy_root()`) and nested TDP roots:**
  Before memslots exist or when a nested root GPA is invalid, `mmu->root.hpa`
  may hold a special dummy root (`KVM_MMU_NO_ROOT_GPA` or `zero_page`, tested
  via `kvm_mmu_is_dummy_root()` in `arch/x86/kvm/mmu/spte.h`) rather than a
  `struct kvm_mmu_page`-backed HPA. Any shadow walker (`shadow_walk_init*()` in
  `arch/x86/kvm/mmu/mmu.c`) must bail out on `!VALID_PAGE(root) ||
  kvm_mmu_is_dummy_root(root)`. When recursively zapping disconnected shadow
  pages, check `!child->root_count` so parentless nested TDP roots still in use
  by a vCPU are preserved.

- **Retry loops (`RET_PF_RETRY`) must reload MMU and top up caches inside the loop:**
  Looping fault/pre-fault callers (such as `kvm_arch_vcpu_pre_fault_memory()`
  in `arch/x86/kvm/mmu/mmu.c`) that retry on `RET_PF_RETRY` cannot hoist
  `kvm_mmu_reload(vcpu)` or `mmu_topup_memory_caches()` outside the loop. A
  concurrent memslot/root invalidation mid-loop will make `is_page_fault_stale()`
  return `RET_PF_RETRY` forever unless `kvm_mmu_reload()` runs on every
  iteration, and an iteration that partially populates upper-level SPTEs before
  retrying can exhaust `mmu_page_header_cache` on the next pass.

- **Atomic SPTE updates must never corrupt `FROZEN_SPTE`:**
  In `arch/x86/kvm/mmu/spte.h`, `FROZEN_SPTE` (`SHADOW_NONPRESENT_VALUE |
  0x1a0ULL`) is a multi-bit non-present sentinel written atomically by threads
  modifying an SPTE under shared `mmu_lock`. Notice bit 5 (`0x20`, the x86
  Accessed bit) is `1` in `FROZEN_SPTE`. Using blind atomic bit-clear (`LOCK
  AND ~spte_accessed_mask`) to age an SPTE clears bit 5 inside `FROZEN_SPTE`,
  breaking `is_frozen_spte()` and corrupting concurrent TDP/Shadow MMU updates.
  Clearing Accessed/Dirty bits on SPTEs that may concurrently transition to
  `FROZEN_SPTE` requires `try_cmpxchg64()` after verifying `!is_frozen_spte()`
  (see `tdp_mmu_clear_spte_bits()` in `arch/x86/kvm/mmu/tdp_mmu.c`). Note:
  this x86 `FROZEN_SPTE` constraint supersedes the generic atomic-AND remark in
  `kvm.md`.

- **Lockless rmap walks and `PREEMPT_RT` bit-spinlocks:**
  - `__kvm_rmap_lock()` in `arch/x86/kvm/mmu/mmu.c` is a bit-spinlock in bit 0
    of `rmap_head->val`. On `!PREEMPT_RT`, holding `mmu_lock` (an `rwlock_t`)
    disables preemption; on `PREEMPT_RT=y`, `rwlock_t` is sleepable and does
    NOT disable preemption, so rmap bit-spinlocks cannot safely run locklessly
    outside `mmu_lock` write exclusion unless `PREEMPT_RT` is accounted for.
  - **Intentional lock elision (do NOT misflag, but enforce value forwarding):**
    `kvm_rmap_lock_readonly()` intentionally returns `0` *without* disabling
    preemption or setting the lock bit when `READ_ONCE(rmap_head->val) == 0`.
    Reviewers should not flag the missing lock on `0` as a bug — **however**,
    the caller MUST pass that exact returned `unsigned long` value into the
    rmap iterator (`for_each_rmap_spte_lockless`) rather than re-reading
    `rmap_head->val`, which would race a concurrent insertion and walk the
    rmap unlocked with preemption enabled.

- **Bounding guest-controlled alias lists:**
  A malicious guest can map the same guest page table frame at thousands of
  GVAs, growing singly-linked `sp->parent_ptes` lists (including on direct
  shadow pages in indirect MMUs) and turning zapping under `mmu_lock` into an
  unpreemptible $O(N^2)$ stall. Keep rmap and `parent_ptes` chains bounded by
  `RMAP_RECYCLE_THRESHOLD`.

**REPORT as bugs:**
- Calling `is_page_fault_stale()` only *before* `make_mmu_pages_available()`
  without re-checking *after* `make_mmu_pages_available()`.
- Walking an x86 MMU root HPA without checking `kvm_mmu_is_dummy_root()`.
- Using atomic `AND` (instead of `cmpxchg` after an `is_frozen_spte()` check)
  to clear bits on x86 SPTEs that can concurrently hold `FROZEN_SPTE`.
- Calling `kvm_rmap_lock_readonly()` and then re-reading `rmap_head->val`
  instead of consuming the value returned by `kvm_rmap_lock_readonly()`.
- `RET_PF_RETRY` loops that invoke `kvm_mmu_reload()` or memory-cache top-up
  only once outside the loop rather than inside the retry loop.

## Exit Handling and Return-Value Contracts

Violating KVM x86's tri-state exit return convention causes infinite guest
VM-Exit loops, corrupted `kvm_run` structures delivered to userspace, swallowed
`-errno` failures, or host panics under `mmu_lock`.

The x86 exit-handler return convention (`handle_exit` in `arch/x86/kvm/x86.c`,
`vmx_handle_exit()`, `svm_invoke_exit_handler()`, `tdx_handle_exit()`) has
strict per-value obligations:

| Return Value | Meaning | Caller / Handler Obligation |
|--------------|---------|-----------------------------|
| `> 0` (`1`) | Handled in kernel; resume guest | Faulting exits MUST either advance guest `RIP` (`kvm_skip_emulated_instruction()`) or queue an architectural exception/VM-Exit (unless retrying a resolved memory/EPT/NPT fault). |
| `0` | Exit to host userspace | `vcpu->run->exit_reason` and its union payload MUST be populated before returning `0` (never leave `KVM_EXIT_UNKNOWN` or stale fields from the previous exit). |
| `< 0` (`-errno`) | Abort `ioctl(KVM_RUN)` with error | Must be preserved up the call stack; post-exit hooks (such as `bus_lock_detected` handling) must NOT overwrite a `< 0` error with `0`. |

- **`-EFAULT` vs `-EIO` and `KVM_BUG_ON()` vs `BUG_ON()`:**
  - In MMU and page-fault paths, returning `-EFAULT` means a resolvable guest
    memory fault occurred and **requires** populating `vcpu->run` via
    `kvm_mmu_prepare_memory_fault_exit()` so `KVM_CAP_EXIT_MEMORY_FAULT` works.
  - Internal KVM invariant violations ("can't happen" states) must NEVER return
    `-EFAULT` and must NEVER call raw `BUG_ON()` (especially under `mmu_lock`,
    which panics the entire host). Use `KVM_BUG_ON(cond, kvm)` and return
    `-EIO`.
- **Guest-supplied bad GPAs in hypercalls / CoCo requests:**
  When a guest passes an unmapped or invalid GPA to a PV hypercall or
  GHCB/TDVMCALL request (e.g., `kvm_read_guest()` fails on an SNP Guest Request
  buffer in `arch/x86/kvm/svm/sev.c`), return the architectural guest error
  code to the guest and return `1` to resume the guest — do NOT return `-EIO`
  (which kills the VM on guest-controlled input). **Exception:** once hardware
  or PSP firmware has irreversibly advanced cryptographic state (such as
  incrementing an SNP VMPCK message sequence number), failing to write the
  response back to guest memory cannot be recovered by the guest and MUST
  terminate the VM (`-EIO`).

**REPORT as bugs:**
- Fault-like VM-Exit instruction handlers (e.g., `VMPTRLD`, `VMPTRST`, MSR/CR
  intercepts) returning `1` on an error/unhandled branch without advancing
  guest `RIP` or injecting an exception.
- Returning `0` to exit to userspace on an error path without setting
  `vcpu->run->exit_reason`.
- Post-exit modifiers (e.g., `bus_lock_detected`) unconditionally returning `0`
  and overwriting a negative `-errno` from the primary exit handler.
- Returning `-EFAULT` for internal KVM invariant bugs (use `KVM_BUG_ON()` +
  `-EIO`), or returning `-EFAULT` from a guest memory fault path without
  `kvm_mmu_prepare_memory_fault_exit()`.
- Using `BUG_ON()` instead of `KVM_BUG_ON()` in KVM runtime/MMU paths.

## Architectural State Validation and Capability Layers

Userspace ioctls (`KVM_SET_SREGS*`, `KVM_SET_NESTED_STATE`, `KVM_SET_MSRS`,
`KVM_SET_CPUID2`) and guest-writable state save areas (`SMRAM` on `RSM`,
`vmcb12`/`vmcs12`) can construct register combinations that native x86
instructions can never produce, violating MMU walker invariants and causing
out-of-bounds array accesses or silent state desynchronization.

x86 KVM distinguishes three capability helpers (`arch/x86/kvm/cpuid.h`); using
the wrong layer breaks `clearcpuid`, module parameters, or per-VM CPUID gating:

| Helper | Meaning | When to Use |
|--------|---------|-------------|
| `boot_cpu_has()` / `cpu_feature_enabled()` | Raw host CPU support | Host-only errata or host hardware paths independent of KVM feature masking. |
| `kvm_cpu_cap_has(feat)` | Supported by both host and KVM | Advertising `KVM_GET_SUPPORTED_CPUID2` and host-initiated MSR bounds. |
| `guest_cpu_cap_has(vcpu, feat)` | Exposed in this vCPU's guest CPUID | Enforcing guest `#GP`/`#UD` faults, guest `DR6`/`CR4`/`EFER` reserved bits, and guest architectural behavior. |

- **Cross-register architectural invariants (`EFER`, `CR0`, `CR4`, `CR8`):**
  - The x86 shadow MMU assumes `EFER.LMA == 1` implies `CR4.PAE == 1` and
    `EFER.LME == 1` (otherwise `paging32_walk_addr_generic()` in
    `arch/x86/kvm/mmu/paging_tmpl.h` walks 4/5-level page tables with a
    2-level array). Every path that loads bulk architectural state —
    `KVM_SET_SREGS*` (`arch/x86/kvm/x86.c`), `KVM_SET_NESTED_STATE`, nested
    `VMRUN`/`VMRESUME`, and `RSM` copying `SMRAM` back into `vmcb01` or L2
    (`arch/x86/kvm/svm/nested.c`, `arch/x86/kvm/vmx/nested.c`) — must validate
    `EFER` (`kvm_valid_efer()`), `CR0`/`CR4`/`CR8` (`kvm_is_valid_sregs()`,
    `kvm_is_valid_cr8()`), and `vmx_emulation_required()` (synthesizing
    architectural shutdown on `RSM` rather than tripping `KVM_BUG_ON`).
  - **Intentional uABI exception (`host_initiated` MSR writes — do NOT flag):**
    Userspace VMMs are permitted to call `KVM_SET_MSRS` *before*
    `KVM_SET_CPUID2`. Therefore, `kvm_set_msr_common()` with
    `msr_info->host_initiated` intentionally checks `kvm_caps.supported_efer_bits`
    and skips guest-CPUID-based `__kvm_valid_efer()` checks, whereas
    `KVM_SET_SREGS*` and guest `WRMSR` enforce `kvm_valid_efer()`.
  - Never infer immutable confidential-VM parameters (such as a TDX TD's EPT
    root level) from mutable userspace-supplied guest CPUID (`MAXPHYADDR`);
    track them explicitly from VM initialization parameters.

- **Defeature polarity, `DR6` active-low vs fixed-1 bits, and TRAP vs FAULT exits:**
  - Watch inverted polarity on AMD defeature bits such as
    `X86_FEATURE_EFER_LMSLE_MBZ` (`1` means `EFER.LMSLE` is unsupported and
    must be zero).
  - Do not conflate active-low `DR6` status bits (`DR6_RTM`, `DR6_BLD`) with
    architecturally fixed-1 bits (`kvm_get_dr6_fixed_1(vcpu)`). `DR6.BLD` is
    active-low (`0` when asserted) and sticky across `#DB` exceptions until
    software clears it; unconditionally OR-ing `DR6_ACTIVE_LOW` clobbers
    previously asserted `DR6.BLD` state, and using a static host `DR6_FIXED_1`
    mask ignores guest CPUID.
  - **TRAP vs FAULT VM-Exits:** Standard MSR/CR intercepts (`SVM_EXIT_MSR`,
    `EXIT_REASON_MSR_WRITE`) are fault-like (hardware exits *before* modifying
    the register). Trap-like exits (such as SEV-ES `SVM_EXIT_WRITE_EFER` /
    `CR_TRAP`) commit the guest's raw value into the VMCB/VMCS *before*
    exiting; masking or rejecting bits only inside `kvm_set_msr()` fails unless
    the VMCB/VMCS field is also sanitized.
  - **Hardware exit-code bits indexing software tables:** Mask hardware
    `#NPF`/EPT error codes to the exact bits encoded in software lookup tables
    (e.g., `permission_fault()`'s 16-entry `permissions[]` table vs hardware
    setting `PFERR_SS_MASK` for shadow-stack accesses).

- **Software vs hardware state synchronization and `vcpu_load()` side effects:**
  - When userspace updates `vcpu->arch.mp_state` (e.g., `KVM_SET_MP_STATE`
    transitioning out of `KVM_MP_STATE_HALTED`), sticky hardware state such as
    `VMCS.GUEST_ACTIVITY_STATE = GUEST_ACTIVITY_HLT` (set directly by guest
    `HLT` when HLT-exiting is disabled) must also be cleared.
  - `vcpu_load()` runs on **every** vCPU ioctl (`KVM_GET_MSRS`, `KVM_GET_SREGS`),
    not only `KVM_RUN`. Any `vcpu_load()` hook that expires timers, injects
    interrupts, or mutates guest-visible MSRs (such as clearing
    `MSR_IA32_TSCDEADLINE` in `apic_timer_expired()` in `arch/x86/kvm/lapic.c`)
    must gate guest-state mutation on `vcpu->wants_to_run` so live-migration
    state save after pCPU migration does not corrupt guest state.

- **TSC scaling, signed time deltas, and timer clamping:**
  - Combining L1 and L2 TSC scaling ratios (`arch/x86/kvm/x86.c`) can overflow
    `u64` or exceed `[1, kvm_caps.max_tsc_scaling_ratio]`. Always clamp/saturate
    against hardware limits; on SVM, writing an out-of-range value to
    `MSR_AMD64_TSC_RATIO` faults via exception fixup and silently leaves the
    *previous* vCPU's TSC ratio active in hardware.
  - Time deltas (`kernel_ns - master_kernel_ns`, `now - state_entry_time`) can
    be negative due to warp or cross-CPU snapshots; guard unsigned
    `mul_u64_u64_shr()` and `u64` runstate accumulators against negative
    deltas.
  - Clamp guest-programmed timer deadlines (e.g., Hyper-V stimer `100 * delta`
    in `arch/x86/kvm/hyperv.c`) before converting to `ktime_t`/`s64`; signed
    overflow produces a deadline in the past, firing `hrtimer` immediately on
    every guest-entry retry and livelocking the host CPU.

**REPORT as bugs:**
- Restoring guest state from `SMRAM` (`RSM`) or userspace ioctls without
  validating `EFER.LMA`/`EFER.LME`/`CR4.PAE` consistency and reserved register
  bits.
- Mutating guest-visible architectural state (such as zeroing
  `MSR_IA32_TSCDEADLINE` or injecting timer events) from `vcpu_load()` when
  `!vcpu->wants_to_run`.
- Converting unbounded guest timer ticks (`u64`) into `s64`/`ktime_t`
  `hrtimer` deadlines without clamping against signed overflow.
- Using `boot_cpu_has()` instead of `kvm_cpu_cap_has()` / `guest_cpu_cap_has()`
  when advertising or enforcing KVM-virtualized CPU features.

## Guest Shared Memory, Atomics, and Confidential Computing

Operating directly on guest-mapped memory (`gfn_to_pfn_cache`, PV structures,
saved hardware GPAs) or passing host buffers to Confidential Computing firmware
without x86 alignment, double-fetch, and page-granularity protections causes
host `#AC` split-lock traps, guest PTE write-protection bypasses, kernel stack
leaks, and host RMP violation panics.

- **Host Split-Lock `#AC` on guest-aligned shared memory:**
  When the host kernel enables split-lock detection, executing a locked atomic
  instruction (`lock btsq`, `lock orq`, `atomic64_*`, `test_and_set_bit()`)
  across a 64-byte cacheline boundary triggers `#AC` in the host kernel.
  Guest-registered PV structures (such as Xen `vcpu_info` in
  `arch/x86/kvm/xen.c`, where 32-bit compat ABI only aligns `evtchn_pending_sel`
  to 4 bytes) cannot use 64-bit `test_and_set_bit()` unless 8-byte alignment is
  enforced at registration time or 32-bit locked ops are used on unaligned
  offsets.

- **Double-fetch of mode flags and saved-GPA TOCTOU:**
  - **Latch dynamic mode flags:** In PV paths where `long_mode` (32-bit vs
    64-bit layout) can be toggled concurrently by another vCPU (e.g.,
    `kvm->arch.xen.long_mode`), read the mode flag **once** into a local
    `bool` before bounds-checking indices and dereferencing union arrays;
    reading it twice allows a bounds check against 4096 ports followed by an
    out-of-bounds index into a 1024-port 32-bit array. Use `READ_ONCE()` /
    `WRITE_ONCE()` on locklessly polled per-vCPU fields.
  - **Saved-GPA write restriction:** Hardware-reported faulting GPAs cached in
    `vcpu->arch.gpa_val` are valid only for the access type of the exit. If
    another vCPU rewrites an MMIO-load instruction into a store before the
    emulator fetches the instruction bytes, reusing `gpa_val` without verifying
    `(error_code & PFERR_WRITE_MASK)` writes through a read-only guest PTE
    (see `emulator_read_write_onepage()` in `arch/x86/kvm/x86.c`).

```c
// WRONG: double-fetch of shinfo mode across validation and array indexing
if (port >= (kvm_xen_has_64bit_shinfo(kvm) ? 4096 : 1024))
	return -EINVAL;
if (kvm_xen_has_64bit_shinfo(kvm)) // may have flipped since check above!
	set_bit(port, shinfo64->evtchn_pending);
else
	set_bit(port, shinfo32->evtchn_pending); // OOB if port >= 1024!

// CORRECT: latch mode once with READ_ONCE() and use it for both steps
bool long_mode = READ_ONCE(kvm->arch.xen.long_mode);
if (port >= (long_mode ? 4096 : 1024))
	return -EINVAL;
```

- **Spectre v1 (`array_index_nospec`), stack bitfield leaks, and GPA overflow:**
  - Any guest- or userspace-controlled array index (such as eVMCS field table
    indices in `arch/x86/kvm/vmx/hyperv_evmcs.h` or MCE `bank` numbers in
    `arch/x86/kvm/x86.c`) must be clamped with `array_index_nospec()` on the
    variable passed to **all** downstream consumers — clamping a local pointer
    while passing the unclamped struct field `mce->bank` to a helper leaves the
    gadget open.
  - Stack structures containing C bitfields copied to guest memory (via
    `kvm_vcpu_write_guest*()`, e.g., Hyper-V SynIC headers) must be
    zero-initialized (`= {}`); assigning a 1-bit field leaves adjacent padding
    bits filled with host kernel stack data.
  - Validate `(gpa, nr_pages)` ioctl regions with `check_shl_overflow(nr_pages,
    PAGE_SHIFT, &nr_bytes)` and `check_add_overflow(gpa, nr_bytes, &end)`
    rather than `gpa + (nr_pages << PAGE_SHIFT) <= gpa` (which wraps silently
    when `nr_pages >= 2^52`), and watch inclusive (`gpa + nr_bytes - 1`) vs
    exclusive (`gpa + nr_bytes`) end checks at `ULLONG_MAX`.

- **Confidential Computing page ownership, GUP, and `guest_memfd` idioms:**
  - **4KB RMP granularity vs `kmalloc()`:** On AMD SEV-SNP hosts, PSP firmware
    commands (such as debug `{DE,EN}CRYPT` in `arch/x86/kvm/svm/sev.c`)
    transition the **entire 4KB physical page's RMP entry** to Firmware-owned.
    Passing a sub-page `kmalloc()` buffer shares that 4KB page with unrelated
    host slab objects and triggers fatal host `RMP violation #PF` panics; always
    allocate full pages (`alloc_page()`).
  - **GUP dirtying and `FOLL_LONGTERM`:** Use `set_page_dirty_lock()`, never
    bare `set_page_dirty()`, on unlocked GUP-pinned pages. When pinning guest
    memory with `FOLL_LONGTERM` (`sev_pin_memory()`), do not combine
    `FOLL_WRITE | FOLL_LONGTERM` unless host CPU writes are actually performed
    through the pin, as `FOLL_WRITE | FOLL_LONGTERM` fails on shared
    file-backed mappings (`virtio-pmem`) while `FOLL_LONGTERM` alone already
    breaks CoW.
  - **Benign CoCo / `guest_memfd` patterns (do NOT flag as bugs):**
    1. `kvm_gmem_get_pfn()` intentionally drops the `struct folio` reference
       before returning the PFN so folio refcounts do not block shared/private
       conversions; lifetime during SPTE installation is protected by
       `mmu_invalidate_seq` + `mmu_lock`, NOT `struct page` refcounts.
    2. During SNP `#NPF` or `PSMASH` handling, observing an unassigned RMP
       entry before checking `mmu_invalidate_retry_gfn()` is a benign race with
       concurrent `guest_memfd` hole-punching / shared conversion, not a bug
       warranting `WARN_ON`.

**REPORT as bugs:**
- Passing sub-page slab allocations (`kmalloc` / `kzalloc`) to SEV/SNP PSP
  firmware operations that modify 4KB RMP ownership.
- Calling `set_page_dirty()` instead of `set_page_dirty_lock()` on unlocked
  GUP-pinned pages.
- Issuing 64-bit locked bitops/atomics (`test_and_set_bit`, `clear_bit`,
  `atomic64_*`) on guest-mapped `gfn_to_pfn_cache` memory whose alignment is
  not guaranteed to be at least 8 bytes.
- Copying stack-allocated structs with bitfields or padding to guest memory
  without `= {}` / `memset` zero-initialization.
- Checking `(gpa + (nr_pages << PAGE_SHIFT) <= gpa)` without
  `check_shl_overflow()` on user-supplied `nr_pages`.

## VCPU, VM, and Vendor Module Teardown

Partial initialization failures, intra-host VM migration, and vendor module
reloads (`kvm_intel.ko` / `kvm_amd.ko` vs `kvm.ko`) leave dangling physical
addresses in hardware-walked tables, use-after-free workqueues, and double-freed
resources when teardown assumes full initialization succeeded.

- **Hardware-walked VM tables on failed `vcpu_create()`:**
  Vendor `vcpu_create()` paths populate VM-wide tables walked directly by
  hardware or peer vCPUs — Intel Posted-Interrupt `pid_table`
  (`arch/x86/kvm/vmx/posted_intr.c`) and AMD AVIC `avic_physical_id_table`
  (`arch/x86/kvm/svm/avic.c`). If a subsequent step in `vcpu_create()` or
  `kvm_vm_ioctl_create_vcpu()` fails, `vcpu_free()` MUST clear that vCPU's
  table entry before freeing `pi_desc` or the APIC backing page.
- **Pre-destroy workqueue cancellation:**
  VM-level delayed work or timers that iterate vCPUs or inject interrupts
  (such as `vioapic->eoi_inject` in `arch/x86/kvm/ioapic.c`) must be cancelled
  synchronously (`cancel_delayed_work_sync()`) in `kvm_arch_pre_destroy_vm()`
  *before* vCPUs are destroyed, not after.
- **SEV intra-host migration and partial-init teardown (`arch/x86/kvm/svm/sev.c`):**
  1. Perform cache maintenance (`sev_writeback_caches()`) on the source VM
     **before** clearing `src->active = false`; concurrent host page reclaim
     checks `sev_guest(kvm)` (`sev->active`) to decide whether to flush guest
     ciphertext before returning pages to the host allocator.
  2. Resources allocated before `sev->active = true` (or retained on a source
     VM after migration clears `sev->active = false`, such as `have_run_cpus`)
     cannot rely on `if (!sev_guest(kvm)) return;` in `sev_vm_destroy()`, and
     init error paths that free pointers early must nullify them (accounting
     for `CPUMASK_OFFSTACK=n` where `cpumask_var_t` is an inline array).
  3. Cross-VM mirror lists (`enc_context_owner`, `mirror_vms`) require
     `sev_mirror_lock`, and `sev_mirror_lock` must be released *before* calling
     `kvm_put_kvm(owner_kvm)` so VM destruction never runs inside the leaf lock.
- **Cross-module state (`kvm.ko` vs `kvm_intel.ko` / `kvm_amd.ko`):**
  `kvm.ko` stays loaded when `kvm_intel` or `kvm_amd` is unloaded and reloaded
  (potentially with different module parameters such as `nested=0`). Any
  `kmem_cache *` destroyed on vendor unload (`mmu_destroy_caches()`) must be
  set to `NULL`, and global masks in `kvm.ko` mutated during vendor load (such
  as `efer_reserved_bits`) must be fully re-initialized on every load.
- **Shadow VMCS pointer visibility vs `VMCLEAR`:**
  When freeing a shadow VMCS (`arch/x86/kvm/vmx/nested.c`), `vmcs01.shadow_vmcs`
  must remain visible until `vmcs_clear()` completes (in case CPU migration
  triggers `VMCLEAR`), and then be set to `NULL` *before* `free_vmcs()`.

**REPORT as bugs:**
- Registering a vCPU pointer or physical page in a VM-wide table during
  `vcpu_create()` without clearing that entry in `vcpu_free()` on partial-init
  failure.
- Clearing a VM's CoCo active flag (`sev->active = false`) before completing
  cache writeback/invalidation on its encrypted pages.
- Destroying `kmem_cache` pointers or mutating globals stored in `kvm.ko` from
  vendor module init/exit without resetting/nullifying them across vendor
  module reloads.
- Calling `kvm_put_kvm()` while holding a leaf lock that `kvm_destroy_vm()` can
  re-acquire.
