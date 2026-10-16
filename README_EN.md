# manet-persistent-root-research

> A **stuck research log**: persistent root on the **Redmi K70 Pro** (`manet` / SM8650 / HyperOS 3.0.9.0 / kernel 6.1.118) with a **locked bootloader**.
>
> **Status: NOT achieved.** Wrong assumptions were corrected into the right ones — but exploitability (reachability) remains unsolved.
> **Contributors welcome.** Every confirmed fact, hard threshold, dead end and open question is documented below.

[![Device](https://img.shields.io/badge/device-manet%20%2F%2023117RK66C-red)]()
[![Kernel](https://img.shields.io/badge/kernel-6.1.118--android14--11-blue)]()
[![CVE](https://img.shields.io/badge/CVE--2026--43499-futex%20PI%20UAF-orange)]()
[![Status](https://img.shields.io/badge/status-blocked-yellow)]()

---

## TL;DR

| Question | Answer |
|---|---|
| Can the bootloader be unlocked? | ❌ No. `sys.oem_unlock_allowed=0`, `ro.boot.flash.locked=1`, `verifiedbootstate=green` |
| Is temporary root available? | ✅ **Yes, 100% reproducible** (§2 — a vendor-left `mqsas` injection channel) |
| Is persistent root available? | ❌ **No.** Lost on every reboot |
| How far did we get? | 🟡 Stack geometry solved, waiter offsets corrected, 3 candidate carriers — **but reachability is unsolved** |
| Self-assessed odds | 10–15% (single GhostLock path: 5–10%) |

---

## The core unsolved problem

> **Geometry ≠ exploitability.**
>
> We proved the buffer **can** overlap `rt_mutex_waiter`.
> We did **not** prove the **kernel will actually dereference the overwritten location**,
> inside the window where the waiter is freed and the stack has been reused.

Reference: the `core_sys_select` wall died on exactly this distinction — it `kvmalloc_node`s the fd bitmap onto the **heap**, so "overlap" is physically impossible there. See [`docs/06-dead-ends.md`](docs/06-dead-ends.md).

**Full write-up (Chinese, with disassembly evidence):**

- [`docs/01-device-facts.md`](docs/01-device-facts.md) — device facts, hardening, writable-node inventory
- [`docs/02-temp-root.md`](docs/02-temp-root.md) — working temp-root recipe
- [`docs/03-waiter-offsets.md`](docs/03-waiter-offsets.md) — **the offset correction, with 3 independent disassembly proofs**
- [`docs/04-stack-geometry.md`](docs/04-stack-geometry.md) — `T0 = SP0 − 0x1E8`, coverage window, hard threshold
- [`docs/05-carriers.md`](docs/05-carriers.md) — candidate carriers + elimination list
- [`docs/06-dead-ends.md`](docs/06-dead-ends.md) — dead ends (don't repeat them)
- [`docs/07-open-questions.md`](docs/07-open-questions.md) — **what we need help with**
- [`docs/08-tuning-annex.md`](docs/08-tuning-annex.md) — battery/perf findings (side product)

---

## Headline results

### 1. The published waiter offsets are **wrong** for this kernel

If your reference says `+0x28 pi_tree` / `+0x50 task` / `+0x58 lock`, that's the `rt_waiter_node` 5-word layout. **It does not apply here.**

| Field | Published | **Measured on this kernel** |
|---|---|---|
| `pi_tree_entry` | +0x28 | **+0x18** |
| `task` | +0x50 | **+0x30** |
| `lock` | +0x58 | **+0x38** |
| `prio` | +0x40 | **+0x44** |
| `deadline` | +0x48 | +0x48 |

Proof: `task_blocks_on_rt_mutex @ 0xffffffc008fff054` → `stp x20, x19, [x21, #0x30]`; `rt_mutex_adjust_prio_chain @ 0xffffffc008fffc88` → `sub x8, x8, #0x18` (the `rb_entry → waiter` back-off proving `pi_tree @ +0x18`). Full evidence in [`docs/03`](docs/03-waiter-offsets.md).

### 2. Stack geometry solved

```
__arm64_sys_futex         0x70
  └─ do_futex             0x60
       └─ futex_wait_requeue_pi  0x1b0
                                 ─────
  waiter at sp+0x98              ⇒  T0 = SP0 − 0x1E8
```

`SP0` = the SP at entry to any `__arm64_sys_*` (all syscalls enter through the same `blr` in `invoke_syscall` ⇒ same value). `CONFIG_RANDOMIZE_KSTACK_OFFSET` is compiled in but **off by default** (static key initial 0) ⇒ **deterministic**.

### 3. Hard threshold (necessary condition)

```
δ = D − off_b ≥ 0x210          (with published offsets)
δ = D − off_b ≥ 0x1D0 = 464    (with measured offsets)
```

Since `off_b ≥ 0`:

> ## ⛔ `D ≥ 0x210` (528 bytes) is an **absolute** necessary condition.
> Any syscall carrier with total stack depth < 528 bytes is **structurally eliminated**, no matter how large the buffer.

### 4. Candidate carriers (independent scan)

| Carrier | Buffer at | Δ = dest − T0 | Size | Coverage |
|---|---|---|---|---|
| **`rt_sigreturn` fpsimd restore** | `SP0−0x280` | **−0x98** | 0x200 | ✅ 100% — **best**, raw `copy_from_user`, no NUL limit |
| **`ioctl` → ashmem `ASHMEM_SET_NAME`** | `SP0−0x228` | **−0x40** | ≤0x100 | ✅ 100% — `strncpy_from_user`, truncates at first `\0` |
| **`sendmsg` msg_sys** | `SP0−0x1C8` | **+0x20** | 0x38 | ⚠️ partial — zero-remainder aligned |

### 5. Known dead ends

| Path | Why it's dead |
|---|---|
| `pselect6` stack overlay | 6.1 `core_sys_select` uses **`kvmalloc_node`** — bitmap on the **heap**, not the stack |
| Honor `SLIDE_PSELECT_RES_ENCODE` workaround | Depends on `res_*` stack buffers that **don't exist** in this kernel |
| `prctl` | `bl capable` is **before** the only `copy_from_user` ⇒ permission-unreachable |
| `get_user` family | Compiled to inline `ldtr` ⇒ **invisible to call-graph scanning** |
| Driver ioctl | `vfs_ioctl → f_op->unlocked_ioctl` is an **indirect call** ⇒ not in the `bl` graph |

---

## A notable side finding (HyperOS hardening)

After privilege escalation, in domain `u:r:su:s0` (`uid=0`, `CapEff=000001ffffffffff`):

```
am / pm / cmd appops / settings   →   Failed transaction (2147483646)
```

**All framework interfaces are blocked even as root** (same when dropping to `su 2000`).
⇒ Only kernel/sysfs nodes, direct PID kills, and `dumpsys` (read-only) remain usable.

---

## How to contribute

**What's most needed: runtime verification of reachability (see [`docs/07`](docs/07-open-questions.md)).**

Concrete directions:

1. **Finish the whole-kernel carrier scan** — must write results to disk incrementally (the previous run buffered output and lost `hits.json`/`dests.json` entirely)
2. **Cover indirect calls** — recover driver ioctl handlers from the `file_operations` data segment
3. **Cover `get_user` inline `ldtr`** — requires dataflow analysis, not call-graph traversal
4. **Runtime validation** — measure the reuse window, locate the dereference site
5. **Alternative targets** — if a better UAF/race exists on this SoC/kernel, please raise it

**Methodology warning**: any scan based on the `bl` call graph will miss **LTO-inlined `_copy_from_user`** and **inline-`ldtr` `get_user`**. That's an inherent blind spot.

---

## Disclaimer

This project is **security research on the author's own device only**. The `manet` bootloader lock was **not** bypassed — **no partition writes, no bootloader modification, no data loss** throughout.

Kernel exploitation research toward persistent root carries a **bricking risk**. Evaluate it yourself. The author accepts no liability for device damage, data loss, or legal consequences arising from use of this material.

---

**Last updated**: 2026-10-16
**Status**: 🟡 blocked — geometry solved, exploitability unsolved, awaiting contributors
