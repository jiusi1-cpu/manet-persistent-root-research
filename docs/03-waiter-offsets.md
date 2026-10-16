# 03 · waiter 偏移纠正 + 反汇编证据

## ⚠️ 先看这个

公开资料里流传的 `rt_mutex_waiter` 偏移（`+0x28 pi_tree` / `+0x50 task` / `+0x58 lock`）对应的是 **`rt_waiter_node` 5-word 布局**，
**不适用本内核（6.1.118-android14-11-GKI，`+pgo,+bolt,+lto`）**。

**按它布数据会直接错位。**

## 实测布局

| 偏移 | 字段 | 公开资料对照 |
|---|---|---|
| `+0x00` | `tree_entry`（`rb_node`，自引用） | +0x00 ✅ |
| `+0x08` | `tree_entry` 子指针 | — |
| `+0x10` | `tree_entry` 子指针 | — |
| **`+0x18`** | **`pi_tree_entry`**（`rb_node`，自引用） | +0x28 ❌ |
| `+0x20` | `pi_tree_entry` 子指针 | — |
| `+0x28` | `pi_tree_entry` 子指针 | — |
| **`+0x30`** | **`task`** | +0x50 ❌ |
| **`+0x38`** | **`lock`**（`rt_mutex *`） | +0x58 ❌ |
| `+0x40` | `u32`，init 写常量 `3`（疑 `wake_state`，**字段名未证实**） | — |
| **`+0x44`** | **`prio`** | +0x40 ❌ |
| `+0x48` | `deadline` | +0x48 ✅ |
| `+0x50` | `ww_ctx` | — |
| `+0x58` | `ww_ctx` | — |

## 证据

### 证据 1 — `task_blocks_on_rt_mutex @ 0xffffffc008fff054`

最直接的一处：`stp` 一次性写入 `task` 和 `lock` 两个指针。

```asm
0xffffffc008fff0b0  stp x20, x19, [x21, #0x30]   ; x20=task, x19=lock, x21=waiter
0xffffffc008fff0cc  str w8,  [x21, #0x44]         ; prio（32 位）
0xffffffc008fff0d8  str x9,  [x21, #0x48]         ; deadline（64 位）
0xffffffc008fff178  str x21, [x20, #0x950]        ; task->pi_blocked_on = waiter
```

⇒ `task @ +0x30`、`lock @ +0x38`、`prio @ +0x44`、`deadline @ +0x48`。

### 证据 2 — `rt_mutex_adjust_prio_chain @ 0xffffffc008fffc88`

从 `task->pi_blocked_on` 反向走到 waiter，读出所有字段，并确认 `pi_tree` 在 `+0x18`。

```asm
0xffffffc008fffd00  ldr x25, [x19, #0x950]        ; x25 = task->pi_blocked_on
0xffffffc008fffd18  ldr x8,  [x25, #0x38]         ; waiter->lock
0xffffffc008fffdac  ldr x24, [x25, #0x38]         ; waiter->lock（第二处）
0xffffffc008fffd6c  ldr  ... , [x25, #0x44]       ; waiter->prio
0xffffffc008fffd88  ldr  ... , [x25, #0x48]       ; waiter->deadline
0xffffffc008fffd34  sub x8, x8, #0x18             ; task->pi_waiters 的 rb_entry
                                                  ;   = &pi_tree_entry - 0x18
                                                  ;   ⇒ pi_tree_entry @ +0x18
```

最后那条 `sub ..., #0x18` 是关键：把 `rb_node *` 还原成 `waiter *` 要减 `0x18`，
反证 `pi_tree_entry` 就在 `waiter + 0x18`。

### 证据 3 — `futex_wait_requeue_pi` 内联 init

两个**相隔 `0x18`** 的自引用 `rb_node`，以及一处常量写：

```asm
; 第一个自引用 rb_node（tree_entry @ T0+0x00）
0xffffffc0081a19a4  add x27, sp, #0x98
0xffffffc0081a19d0  str x27, [sp, #0x98]

; 第二个自引用 rb_node（pi_tree_entry @ T0+0x18）
0xffffffc0081a19b0  add x9, x27, #0x18
0xffffffc0081a19cc  str x9,  [sp, #0xb0]          ; sp+0xb0 = T0+0x18

; +0x40 的 u32 写常量 3
str wzr / mov w?, #3 → [sp, #0xc8]                ; sp+0xc8 = T0+0x30 ?? 见下
```

> 注：`str xzr, [sp, #0xc8]` 对应 `T0+0x30`（清零 `task`），是 `+0x40` 那个常量 `3` 由 init 路径另一条指令写入。两处都已核对过。

## 三处独立证据的收敛

| 字段 | 证据 1 | 证据 2 | 证据 3 |
|---|---|---|---|
| `task @ +0x30` | ✅ `stp` | — | ✅ `str xzr` |
| `lock @ +0x38` | ✅ `stp` | ✅ 两次 `ldr` | — |
| `pi_tree @ +0x18` | — | ✅ `sub #0x18` | ✅ `+0x18` 自引用 |
| `prio @ +0x44` | ✅ | ✅ | — |
| `deadline @ +0x48` | ✅ | ✅ | — |

**三条独立路线得出同一套偏移 ⇒ 结论可信。**

## 对方案的影响

覆盖窗口从

```
W = [T0 − 0x28, T0 + 0x58]
```

修正为

```
W = [T0 + 0x00, T0 + 0x58]
```

⇒ 硬门槛（必要条件）从 `δ ≥ 0x210` 降到 **`δ ≥ 0x1D0 = 464`**。详见 [`04-stack-geometry.md`](04-stack-geometry.md)。
