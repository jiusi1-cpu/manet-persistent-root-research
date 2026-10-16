# 07 · 待解问题（求接手）

**这是本仓库最需要人手的一部分。** 下面是按优先级排列的开放问题。

---

## 🔴 P0：可执行性（reachability）—— **这才是命门**

> ### 几何成立 ≠ 漏洞可用
>
> 我们证明的是：缓冲区**能**盖住 `rt_mutex_waiter`。
> 我们**没有**证明：**内核会去读被覆盖的那个位置**，并且是在 waiter 已 free、栈已复用的窗口内。

**参考对照**：`core_sys_select` 那一堵墙正是死在这个区别上 —— 位图在堆上，"盖住"从物理上就不成立。
（见 [`06-dead-ends.md`](06-dead-ends.md) §1）

### 需要回答

1. **窗口长度**：从 `futex_wait_requeue_pi` free 掉 waiter，到**同线程**后续 syscall 复用该栈区间，
   窗口有多长？能否稳定撞上？
2. **解引用点**：free 之后，**谁**会去解引用那个 `rt_mutex_waiter`？
   - `rt_mutex_adjust_prio_chain` 的遍历？
   - `task->pi_blocked_on`（`task + 0x950`）的清理路径？
   - 这个解引用点的**时序是否落在窗口内**？
3. **写原语**：覆盖 `lock @ +0x38` 之后，能否让 `rt_mutex` 走到期望的写原语，
   而不是直接 panic（例如解引用伪造的 `lock` 导致 oops）？
4. **竞态稳定性**：需要几个线程、怎样的时序，才能把窗口稳定命中？失败时是干净退出还是 panic？

**验证方式**：需要在真机上做（本仓库未做任何运行时验证）。建议先用 qemu / 同内核 GKI 镜像起一个可复现环境。

---

## 🟡 P1：方法学缺口（把扫描补全）

1. **重跑全内核候选扫描** —— 必须改成**分阶段落盘**。
   现有 `scan.py` / `dests.py` 因输出被缓冲 + 任务中断，**结果全丢**。
   预期产出：`hits.json`（syscall → sink 路径 + 深度）、`dests.json`（`x0` 溯源到 `sp+off`、`x2` 长度推断）。
2. **补间接调用路径** —— 从 `file_operations` 数据段恢复 handler，
   把 `vfs_ioctl → f_op->unlocked_ioctl` 纳入扫描。
3. **补 `get_user` 内联 `ldtr`** —— 需要**数据流分析**，不是调用图遍历。
   目前所有 `get_user` 载体都在盲区里。
4. **补 LTO 内联的 `_copy_from_user`** —— 按真正的 sink `__arch_copy_from_user @ 0xffffffc008fb9300` 重建索引。

---

## 🟡 P2：几何细节

1. **`rt_sigreturn` 的 0x200 拷贝属于哪条分支**
   - `restore_fpsimd_context` @ `0x1f244`，还是
   - SVE 分支 @ `0x1f5e4`？
   两条副本**同址**。SVE 分支依赖的 cpucap 位（bit 51）在 SM8650 上**是否可用未验证**
   （该 SoC 很可能无 SVE，但 fpsimd 主路径不依赖它）。
2. **el0 入口链向量 stub 帧大小**
   相邻项实测 `sub sp, sp, #0x150`，教科书值 `0x110`。
   对**所有候选是同加项**，做差抵消 ⇒ 不影响几何判定，但影响绝对深度 `D` 的报告值。

---

## 🟢 P3：换靶点 / 其他方向

1. **枚举厂商遗留通道** —— `mqsas` 那条路说明厂商会留系统服务接口。
   还有哪些 MIUI/HyperOS 服务能被类似方式调用？权限边界在哪？能否在开机流程里自动触发？
   （**这可能是比内核漏洞更现实的持久化路径**）
2. **找更适合的 UAF / 竞态** —— 本 CVE 在 6.1 上被 `core_sys_select` 的堆分配堵死。
   如果 SM8650 / kernel 6.1.118 上有别的候选，欢迎提。
3. **官方解锁策略变化跟踪** —— 目前 `sys.oem_unlock_allowed=0` 是机型策略锁死，
   但如果小米后续放开（或社区找到绕过），这条路成本最低。

---

## 已有产出（可复用）

| 路径 | 内容 | 状态 |
|---|---|---|
| `ghostlock\work\kernel` | 原始未压缩 ARM64 `Image`（35 MB） | 只读，未改 |
| `ghostlock\analysis\kallsyms.txt` | 101,454 个符号 | — |
| `ghostlock\analysis\vmlinux` | — | — |
| `ghostlock\scan\build_index.py` | 全 `.text` 线性反汇编 + 指令分类索引 | ✅ 跑通：4,227,072 条指令 / 55,662 FUNC |
| `ghostlock\scan\cls.bin` / `val.bin` / `br.bin` / `funcs.txt` | 索引产物 | ✅ |
| `ghostlock\scan\scan.py` | 调用图 + 栈深度累加 + 路径枚举 | ❌ 未跑完 |
| `ghostlock\scan\dests.py` | copy 目标是否栈地址 + off + 长度 | ❌ 未运行 |
| `ghostlock\scan\kdump.py` / `kres.py` | 反汇编 / 符号解析工具 | ✅ |
| `ghostlock\scan\apc.txt` / `tbr.txt` | `rt_mutex_adjust_prio_chain` / `task_blocks_on_rt_mutex` 全量反汇编转储 | ✅ |
| `ghostlock\work\kdis.py` | 符号解析 + 反汇编 + 紧凑视图 | ✅ |
| `ghostlock\work\probe.py` | 按函数抽取帧大小与 copy 目标 | ✅ |

> `hits.json` / `dests.json` **不存在**。

---

## 想接手？

提 Issue 说明你想攻哪一条，或直接 PR。
**最有价值的贡献是 P0 的运行时验证结果** —— 哪怕只是"窗口测出来是 X 微秒"或"这里会 panic"，都是决定性信息。
