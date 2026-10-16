# manet-persistent-root-research

> 一个**卡住的研究**：Redmi K70 Pro（`manet` / SM8650 / HyperOS 3.0.9.0 / kernel 6.1.118）在 **bootloader 锁定**条件下的持久化 root 探索。
>
> 当前状态：**未突破**。已把"错的方向"纠正为"对的方向"，但离可用还隔一层。
> **欢迎有兴趣的开发者接手探路** —— 下文把已确认的事实、硬门槛、失败路径和开放问题全部摊开。

[![Device](https://img.shields.io/badge/device-manet%20%2F%2023117RK66C-red)]()
[![Kernel](https://img.shields.io/badge/kernel-6.1.118--android14--11-blue)]()
[![CVE](https://img.shields.io/badge/CVE--2026--43499-futex%20PI%20UAF-orange)]()
[![Status](https://img.shields.io/badge/status-blocked-yellow)]()

---

## TL;DR

| 问题 | 结论 |
|---|---|
| Bootloader 能解锁吗 | ❌ 不能。`sys.oem_unlock_allowed=0`，`ro.boot.flash.locked=1`，`verifiedbootstate=green` |
| 临时 root 有吗 | ✅ **有，且 100% 可复现**（见 §2，厂商遗留的 `mqsas` 注入通道） |
| 持久 root 有吗 | ❌ **没有**。重启即失效 |
| 走到哪一步了 | 🟡 栈几何算通、等待者偏移已纠正、3 个候选载体；**但可执行性（reachability）未解** |
| 自评成功概率 | 10~15%（单条 GhostLock 路线 5~10%） |

---

## 1. 设备事实（全部实测）

```
型号        Redmi K70 Pro (manet / 23117RK66C)
SoC         SM8650  Snapdragon 8 Gen 3
系统        HyperOS  OS3.0.9.0.WNMCNXM
内核        6.1.118-android14-11-gca0ef6d17716-ab13624819
KMI         android14-6.1
内核编译    +pgo,+bolt,+lto
SELinux     Enforcing
```

**锁定状态证据**

```
ro.boot.flash.locked       = 1
ro.boot.verifiedbootstate  = green
sys.oem_unlock_allowed     = 0     ← 官方解锁路径被彻底关闭，不是"等就行"
```

**几个影响研究方法的关键实测事实**

- `ro.boot.selinux` 读出来恒为 `enforcing`，**即使 fastboot 传了 `androidboot.selinux=permissive`**。可靠的判断方式是 `getenforce`。
- `CONFIG_RANDOMIZE_KSTACK_OFFSET` 编译进去了，但**默认关闭**（`static_key_false`，初值 0）⇒ 栈地址是确定的，几何可静态推导。
- `_copy_from_user` 被 **LTO 完全内联**，kallsyms 里没有这个符号。真正的 sink 是 `__arch_copy_from_user @ 0xffffffc008fb9300`。
- `get_user` 系列是内联 `ldtr`，**不进调用图** ⇒ 任何基于 `bl` 图的扫描方法天然漏掉这一类。
- `kptr_restrict = 2`：shell 域读不到 `/proc/kallsyms`；只有提权后的进程可以。

---

## 2. 已完成的：临时 root（可用）

**这是厂商遗留的注入通道，不是漏洞利用，所以稳定。**

```
① fastboot reboot bootloader
② fastboot oem set-gpu-preemption 0 androidboot.selinux=permissive
③ fastboot continue
④ 等 sys.boot_completed=1
⑤ service call miui.mqsas.IMQSNative 21 \
     i32 1 s16 '/data/local/tmp/ksud' \
     i32 1 s16 'late-load' \
     s16 '<outfile>' i32 120
```

注入进程拿到 `uid=0(root) gid=1000(system) context=u:r:hypsys_ssi_default:s0`。

**关键约束**

- 注入**只在 SELinux Permissive 期间有效**。注入的 `late-load` 会触发模块 `service.sh` 把 SELinux 翻回 Enforcing（例如 `h_reboot/service.sh` 里有 `if Permissive then setenforce 1`）。
- Permissive 窗口实测：启动后**稳定 ≥ 50 秒**。
- `ksud`（KernelSU）版本 `3.1.0-29-gf0615d3c`，MD5 `06FE421B10A7BB433B8C6A52F8B20C6D`。
- 恢复路径 100% 验证过，**不丢数据**。

> ⚠️ 这条路重启即失效。它的存在恰恰说明：**能持久化的东西，厂商自己必须留一条"不进 fastboot 也能起来"的路**——值得顺着这个思路找。

---

## 3. 持久化路线：GhostLock / CVE-2026-43499

### 3.1 靶点

`futex_wait_requeue_pi` 路径上的 **PI waiter 栈对象 UAF**（futex PI stack-UAF）。

可行性初筛（已通过）：

```
__arm64_sys_futex
  └─ bl do_futex                     ← 未内联
       └─ bl futex_wait_requeue_pi   ← 未内联
```

符号存在、未被 PGO 内联，路径可达。

### 3.2 ⚠️ 公开资料里的 waiter 偏移在本内核上是**错的**

如果你手上拿到的方案写的是 `+0x28 / +0x50 / +0x58`，**那是 rt_waiter_node 5-word 布局，不适用本内核**。按它布数据会直接错位。

**本内核实测布局（三处独立反汇编互证）**

| 字段 | 公开资料 | **本内核实测** |
|---|---|---|
| `tree_entry` | +0x00 | +0x00 |
| `pi_tree_entry` | +0x28 | **+0x18** |
| `task` | +0x50 | **+0x30** |
| `lock` | +0x58 | **+0x38** |
| `u32`（init 写常量 3，字段名未证实） | — | +0x40 |
| `prio` | +0x40 | **+0x44** |
| `deadline` | +0x48 | +0x48 |
| `ww_ctx` | — | +0x50, +0x58 |

**证据 1** — `task_blocks_on_rt_mutex @ 0xffffffc008fff054`

```asm
0xffffffc008fff0b0  stp x20, x19, [x21, #0x30]   ; x20=task, x19=lock, x21=waiter
0xffffffc008fff0cc  str w8,  [x21, #0x44]         ; prio
0xffffffc008fff0d8  str x9,  [x21, #0x48]         ; deadline
0xffffffc008fff178  str x21, [x20, #0x950]        ; task->pi_blocked_on
```

**证据 2** — `rt_mutex_adjust_prio_chain @ 0xffffffc008fffc88`

```asm
0xffffffc008fffd00  ldr x25, [x19, #0x950]        ; task->pi_blocked_on
0xffffffc008fffd18  ldr x8,  [x25, #0x38]         ; waiter->lock
0xffffffc008fffdac  ldr x24, [x25, #0x38]
0xffffffc008fffd6c  ldr  ... [x25, #0x44]         ; prio
0xffffffc008fffd88  ldr  ... [x25, #0x48]         ; deadline
0xffffffc008fffd34  sub x8, x8, #0x18             ; task->pi_waiters rb_entry → pi_tree @0x18
```

**证据 3** — `futex_wait_requeue_pi` 内联 init：两个相隔 `0x18` 的自引用 `rb_node`，`str xzr, [sp, #0xc8]` = `T0+0x30`。

### 3.3 栈几何（已推导）

锚点 `SP0` = 任一 `__arm64_sys_*` 进入时的 SP。所有 syscall 由 `invoke_syscall` 同一处 `blr` 进入，**值相同** ⇒ 跨 syscall 可比较。

帧累加：

```
__arm64_sys_futex        0x70
  + do_futex             0x60
  + futex_wait_requeue_pi 0x1b0
  ─────────────────────────────
     waiter = 该函数内 sp+0x98
```

⇒ **`T0 = SP0 − 0x1E8`**（488 字节）

`T0` 的两条独立证据：

```asm
; (a) 两个相隔 0x18 的自引用 rb_node
0xffffffc0081a19a4  add x27, sp, #0x98
0xffffffc0081a19d0  str x27, [sp, #0x98]
0xffffffc0081a19b0  add x9, x27, #0x18
0xffffffc0081a19cc  str x9,  [sp, #0xb0]

; (b) futex_q 的 rt_waiter@0x58 / requeue_pi_key@0x60
0xffffffc0081a19f8  stp x27, x28, [sp, #0x60]     ; x28 = sp+0x78 = &key2
0xffffffc0081a19e0  add x3, sp, #8                ; q = sp+8
```
⇒ `sp+0x98` 即 `rt_waiter`。

**覆盖窗口**

```
按公开偏移 : W = [T0−0x28, T0+0x58] = [SP0−0x210, SP0−0x190]
按实测偏移 : W = [T0+0x00, T0+0x58] = [SP0−0x1E8, SP0−0x190]
```

### 3.4 硬门槛（必要条件）

设 `D` = 从 syscall 入口到 copy 站的累计栈消耗，`off_b` = 缓冲区在栈帧内的偏移（`off_b ≥ 0`）：

```
δ = D − off_b ≥ 0x210          （按公开偏移）
δ = D − off_b ≥ 0x1D0 = 464    （按实测偏移）
且 size ≥ δ − 0x190
```

因为 `off_b ≥ 0`：

> **`D ≥ 0x210`（528 字节）是绝对必要条件。**
> 总栈深不足 528 字节的 syscall 载体，**无论 buffer 开多大都结构性出局。**

这道门槛直接把候选砍掉一大片，是筛选时最先该用的尺子。

### 3.5 三个候选载体（独立扫描所得）

| 载体 | 缓冲区位置 | Δ = dest − T0 | 长度 | 覆盖 | 备注 |
|---|---|---|---|---|---|
| **`rt_sigreturn` fpsimd 恢复** | `SP0−0x280` | **−0x98** | 0x200 | ✅ 100% | **首选**，原始 `copy_from_user`，无 NUL 限制 |
| **`ioctl` → ashmem `ASHMEM_SET_NAME`(0x41007701)** | `SP0−0x228` | **−0x40** | ≤0x100 | ✅ 100% | `strncpy_from_user`，**遇首个 `\0` 截断** |
| **`sendmsg` msg_sys** | `SP0−0x1C8` | **+0x20** | 0x38 | ⚠️ 部分 | 零余数对齐，正好压 task/lock/prio/deadline/ww_ctx + pi_tree 两个子指针 |
| `select` | — | −0x08 | ≤0x28 | ❌ | 只够 tree_entry |
| `pselect6` | — | −0x18 | ≤0x28 | ❌ | 只够 tree_entry |

**出局清单**：`poll` −0x24C、`ppoll` −0x26C、`recvmsg` +0x98、`recvmmsg` −0x148、`splice` +0x178、`vmsplice` +0x78、`setsockopt` +0xF0、`binder` −0x60、`tee`（无）。

### 3.6 ⛔ 已知的死路（别重复踩）

| 路径 | 为什么死 |
|---|---|
| **`pselect6` 栈覆盖** | 6.1 的 `core_sys_select` 用 **`kvmalloc_node` 把位图放堆上**，不在栈上 ⇒ pselect 的栈覆盖够不到被 free 的 waiter。**这是最根本的一堵墙。** |
| **Honor `SLIDE_PSELECT_RES_ENCODE` 变通** | 依赖 6.1 的 `res_*` 栈缓冲，**本内核不存在这些结构** |
| **`prctl`** | `bl capable`(@0xd10c4) 确认前置在唯一的 `copy_from_user`(@0xd1320) 之前，几何 Δ=+0x08/≤0x170 但**权限不可达**；且本 build 里连 `PR_SET_MM_MAP` 的 0x60 字节拷贝都不存在 |
| **`get_user` 系列** | 内联 `ldtr`，不在 `bl` 调用图里，扫描天然漏掉 |
| **驱动 ioctl 路径** | `vfs_ioctl → f_op->unlocked_ioctl` 是**间接调用**，不在 `bl` 图里；需另从 `file_operations` 数据段恢复 |

### 3.7 核心未解问题 ← **这是最需要人手的地方**

> **几何成立 ≠ 漏洞可用。**
>
> 我们证明的是"缓冲区**能**盖住 `rt_mutex_waiter`"。
> 我们**没有**证明：**内核会去读被覆盖的那个位置**，并且是在 waiter 已 free、栈已复用的窗口内。

参考对照：`core_sys_select` 那一堵墙正是死在这个区别上 —— 位图在堆上，"盖住"这件事从物理上就不成立。

**需要回答的问题：**

1. 从 `futex_wait_requeue_pi` free 掉 waiter，到同线程后续 syscall 复用该栈区间，**窗口有多长**？能不能稳定撞上？
2. free 之后，**谁**会去解引用那个 `rt_mutex_waiter`？（`rt_mutex_adjust_prio_chain` 的遍历？`task->pi_blocked_on` 的清理？）这个解引用点的时序是否落在窗口内？
3. 覆盖 `lock @+0x38` 之后，能不能让 `rt_mutex` 走到我们期望的写原语上（而不是直接 panic）？
4. 三个载体里，`rt_sigreturn` 的 0x200 拷贝属于 `restore_fpsimd_context`(@0x1f244) 还是 SVE 分支(@0x1f5e4)？两条副本同址，**SVE 分支的 cpucap 位(bit 51) 在 SM8650 上是否可用未验证**（该 SoC 很可能无 SVE，fpsimd 主路径不依赖它）。
5. el0 入口链的向量 stub 帧大小没钉死（相邻项实测 `sub sp, sp, #0x150`，教科书值 `0x110`）。对所有候选是**同加项**，做差抵消，不影响几何判定，但影响绝对深度 `D`。

---

## 4. 交叉验证情况

两条独立路线（几何推导 / 载体扫描）产出的结果**互相印证**：

- 几何线给出硬门槛 `D ≥ 0x210`
- 载体线独立测出 `rt_sigreturn` 缓冲区位于 `SP0−0x280`（深 640 B）、`ioctl` 位于 `SP0−0x228`（深 552 B）
- 两者都**大于** 528 B 门槛 ⇒ ✅ 通过
- 两侧**独立得出同一套偏移修正**（`pi_tree@0x18` / `lock@0x38`）⇒ 可信度较高

---

## 5. 仓库结构

```
.
├── docs/
│   ├── 01-device-facts.md        # 设备事实与锁定状态
│   ├── 02-temp-root.md           # 临时 root 完整配方（可用）
│   ├── 03-waiter-offsets.md      # waiter 偏移纠正 + 反汇编证据
│   ├── 04-stack-geometry.md      # T0 推导、覆盖窗口、硬门槛
│   ├── 05-carriers.md            # 候选载体清单 + 出局清单
│   ├── 06-dead-ends.md           # 已知死路（别重复踩）
│   ├── 07-open-questions.md      # 待解问题（求接手）
│   └── 08-tuning-annex.md        # 附：性能/续航调优实测
├── README_EN.md
└── LICENSE
```

---

## 6. 怎么贡献

**最缺人手的是 §3.7 的可执行性验证。**

具体可接手的方向：

1. **把全内核候选扫描跑完** —— 现有脚本有方法缺陷（见下），需要改成**边跑边落盘**（之前用 `Select-Object -Last` 缓冲输出 + 任务中断，导致 `hits.json`/`dests.json` 从未生成）
2. **补间接调用路径** —— 从 `file_operations` 数据段恢复驱动 ioctl handler，把 `vfs_ioctl` 那条路纳入扫描
3. **补 `get_user` 内联 `ldtr`** —— 需要做数据流分析而非调用图
4. **运行时验证** —— 窗口长度实测、解引用点定位
5. **换靶点** —— 如果有更合适的 UAF/竞态，欢迎提

**方法论提醒**：任何基于 `bl` 调用图的扫描，都会漏掉 `LTO 内联的 _copy_from_user` 和 `内联 ldtr 的 get_user`。这是本方法的固有盲区。

---

## 7. 免责声明

本项目**仅用于本人自有设备的安全研究**。`manet` 的 bootloader 锁定状态未被绕过，**未做任何写分区/改 bootloader 操作，全程不丢数据**。

面向持久化 root 的内核漏洞利用研究存在**设备变砖风险**，请自行评估。作者不对任何因使用本项目内容导致的设备损坏、数据丢失或法律问题负责。

---

## 附：性能/续航调优实测（同机副产品）

顺手做的调优研究，实测数据供参考 —— 详见 [`docs/08-tuning-annex.md`](docs/08-tuning-annex.md)。

**三个反直觉的实测发现：**

1. **电池是硬天花板**：`charge_full 3320 mAh / design 5000 mAh` = **66% 健康度，循环 1751 次**。软件调优补不回丢掉的三分之一。
2. **耗电第一来源不是温控，是内存**：温度实测 35–38 °C、`throttling=0`（完全没热）。但 `zram 累计换出 15.4M 页(≈61 GB)` ⇒ **CPU 单项吃 629 mAh，占总放电 48%**。
3. **小米云控控制的那些调频节点是只读的**：从 `com.xiaomi.joyose` 的 `teg_config.db` 解出云控配置，确认它写的正是 `walt/target_load_thresh` 等节点 —— 而这些节点权限是 **`-r--r--r--`（0444），root 也写不进**，只有小米 `system` 用户能写。任何"性能模块"里对它们的写入都是静默失败的。

**另一条实测边界**：`am` / `pm` / `appops` 在 `u:r:su:s0` 域被 HyperOS 拦截（`Failed transaction 2147483646`）⇒ 提权后**不能**用框架接口，只能用内核节点 + 直接杀进程。

---

**最后更新**：2026-10-16
**状态**：🟡 blocked — 几何已解，可执行性未解，等待接手
