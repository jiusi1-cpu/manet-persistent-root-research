# 04 · 栈几何推导与硬门槛

## 锚点 SP0

`SP0` = 任一 `__arm64_sys_*` **进入时**的 SP。

**为什么跨 syscall 可比**：所有 syscall 由 `invoke_syscall` 的**同一处 `blr`** 进入，
所以在同一线程、无栈随机化的前提下，不同 syscall 的入口 SP 值**相同**。

而本内核 `CONFIG_RANDOMIZE_KSTACK_OFFSET` **默认关闭**（见 `01-device-facts.md`）⇒ 该前提成立。

## 帧累加 ⇒ T0

```
__arm64_sys_futex           0x70
  └─ do_futex               0x60
       └─ futex_wait_requeue_pi   0x1b0
                                  ─────
                                  0x280 累计

waiter 在该函数内位于 sp + 0x98
```

```
T0 = SP0 − (0x70 + 0x60 + 0x1b0) + 0x98
   = SP0 − 0x280 + 0x98
   = SP0 − 0x1E8
```

### **`T0 = SP0 − 0x1E8`（即入口栈顶下方 488 字节）**

---

## T0 的两条独立证据

### 证据 (a)：两个相隔 0x18 的自引用 rb_node

```asm
0xffffffc0081a19a4  add x27, sp, #0x98
0xffffffc0081a19d0  str x27, [sp, #0x98]          ; tree_entry 自引用
0xffffffc0081a19b0  add x9,  x27, #0x18
0xffffffc0081a19cc  str x9,  [sp, #0xb0]          ; pi_tree_entry 自引用 (+0x18)
```

`sp+0x98` 和 `sp+0xb0` 相差正好 `0x18`，与 [`03`](03-waiter-offsets.md) 的布局一致。

### 证据 (b)：`futex_q` 字段位置反推

```asm
0xffffffc0081a19f8  stp x27, x28, [sp, #0x60]     ; x28 = sp+0x78 = &key2
0xffffffc0081a19e0  add x3, sp, #8                ; q = sp + 8
```

6.1 `futex_q` 布局中 `rt_waiter @ 0x58`、`requeue_pi_key @ 0x60`：

```
q          = sp + 0x08
q.rt_waiter         = sp + 0x08 + 0x58 = sp + 0x60
q.requeue_pi_key    = sp + 0x08 + 0x60 = sp + 0x68
```

`sp+0x98` 落在这个结构之后，与 `rt_waiter` 的对齐关系自洽，
且 `stp x27, x28, [sp, #0x60]` 正是把 `&key2` 写进 `q` 的对应位置。

**小结**：帧大小复核全部吻合 —— `0x70` / `0x60` / `0x1b0`。

---

## 覆盖窗口

```
按公开偏移 : W = [T0 − 0x28, T0 + 0x58] = [SP0 − 0x210, SP0 − 0x190]
按实测偏移 : W = [T0 + 0x00, T0 + 0x58] = [SP0 − 0x1E8, SP0 − 0x190]
```

（窗口右端取 `+0x58` 是因为 `ww_ctx` 在 `+0x50`/`+0x58`。）

---

## 硬门槛（必要条件）

定义：

- `D` = 从 **syscall 入口**到 **copy 站**的累计栈消耗
- `off_b` = 缓冲区在该栈帧内的偏移，**`off_b ≥ 0`**
- 则缓冲区起始位置 = `SP0 − (D − off_b)`

要够到窗口左端：

```
SP0 − (D − off_b)  ≤  SP0 − 0x210
⇒  D − off_b  ≥  0x210
⇒  δ = D − off_b ≥ 0x210              （按公开偏移）
⇒  δ ≥ 0x1D0 = 464                     （按实测偏移）
且 size ≥ δ − 0x190
```

因为 **`off_b ≥ 0`**，所以：

> ## ⛔ `D ≥ 0x210`（528 字节）是**绝对必要条件**
>
> **总栈深不足 528 字节的 syscall 载体，无论 buffer 开多大，都结构性地出局。**

这道门槛的价值：它**不需要**跑完整扫描就能砍掉一大片候选，是筛选时该最先用的尺子。

---

## 载体侧结果（独立扫描所得）

三个候选实测位置，**与门槛对照**：

| 载体 | 缓冲区起始 | 深度 D | 过 528 门槛？ |
|---|---|---|---|
| `rt_sigreturn` fpsimd 恢复 | `SP0 − 0x280` | 640 B | ✅ |
| `ioctl` → ashmem `ASHMEM_SET_NAME` | `SP0 − 0x228` | 552 B | ✅ |
| `sendmsg` msg_sys | `SP0 − 0x1C8` | 456 B | ❌ 差 72 B |

**两条独立路线互相印证**：几何线的门槛 `528 B` 与载体线实测的 `640 B` / `552 B` 对照 —— 前两个通过，第三个结构性出局。

详见 [`05-carriers.md`](05-carriers.md)。

---

## 遗留的不确定项

1. **el0 入口链的向量 stub 帧大小没完全钉死**
   相邻项实测 `sub sp, sp, #0x150`，教科书取值 `0x110`。
   对**所有候选是同一个加项**，做差时抵消 ⇒ **不影响几何判定**，但影响绝对深度 `D` 的报告值。

2. **`rt_sigreturn` 的 0x200 拷贝属于哪条分支未定**
   `restore_fpsimd_context`(@`0x1f244`) 还是 SVE 分支(@`0x1f5e4`)，两条副本**同址**。
   SVE 分支依赖的 cpucap 位（bit 51）在 SM8650 上是否可用**未验证** —— 该 SoC 很可能无 SVE，但 fpsimd 主路径不依赖它。

3. **未做任何运行时验证** ⇒ 上述全部为**静态推导**。
