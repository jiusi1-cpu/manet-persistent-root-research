# 01 · 设备事实与锁定状态

全部为**本机实测**，非资料转述。

## 设备身份

| 项 | 值 |
|---|---|
| 型号 | Redmi K70 Pro |
| 代号 | `manet` |
| 机型号 | `23117RK66C` |
| SoC | SM8650 / Snapdragon 8 Gen 3 |
| 序列号 | `cc2c325a` |

## 系统 / 内核

| 项 | 值 |
|---|---|
| ROM | HyperOS `OS3.0.9.0.WNMCNXM` |
| 内核 | `6.1.118-android14-11-gca0ef6d17716-ab13624819` |
| KMI | `android14-6.1` |
| 内核编译选项 | `+pgo,+bolt,+lto` |
| SELinux | `Enforcing` |
| 内核镜像 | 原始未压缩 ARM64 `Image`（35,363,328 B），**file offset == vaddr − 0xffffffc008000000** |

## 锁定状态（决定性）

```
ro.boot.flash.locked       = 1
ro.boot.verifiedbootstate  = green
sys.oem_unlock_allowed     = 0
```

**结论**：官方解锁路径被彻底关闭。`sys.oem_unlock_allowed=0` 意味着这不是"等社区资格"能解决的，是机型策略层面锁死的。

## 影响研究方法的实测事实

### SELinux 与 fastboot 参数

- `ro.boot.selinux` **恒读出 `enforcing`**，即使 fastboot 传入了 `androidboot.selinux=permissive`。
- 判断真实状态的可靠方式是 **`getenforce`**，不是读 prop。
- **Permissive 窗口实测：启动后稳定 ≥ 50 秒。**
- 早些时候观察到"已 Enforcing"的现象，事后确认是**一次 panic 重启丢掉了 boot 参数**导致的，不是窗口本身短。

### 内核加固配置

- `CONFIG_RANDOMIZE_KSTACK_OFFSET` **已编译进去，但默认关闭**：
  实现为 `static_key_false`，静态键初值 0，位于 `__start_ro_after_init` 首对象。
  ⇒ **栈地址确定，几何可静态推导。** 仅在 cmdline 带 `on` 时启用（位移 0..2032，步长 16）。
- `kptr_restrict = 2`：shell 域读不到 `/proc/kallsyms`；**只有提权后的进程可以**（读出 471,903 行符号，地址被置零需另解）。
- SELinux 相关：`setenforce 0` 会被自动翻回；`/sys/fs/selinux/permissive` 节点**不存在**（`CONFIG_SECURITY_SELINUX_DEVELOP` 关闭）。

### 符号与调用图陷阱

| 事实 | 影响 |
|---|---|
| `_copy_from_user` 被 **LTO 完全内联**，kallsyms 无此符号 | 真正的 sink 是 `__arch_copy_from_user @ 0xffffffc008fb9300`，调用图扫描会漏 |
| `get_user` 系列是**内联 `ldtr`** | 不进调用图，**任何基于 `bl` 图的扫描都天然漏掉这一类** |
| `__arm64_sys_futex → do_futex → futex_wait_requeue_pi` | 三级均 `bl`，未被 PGO 内联 ⇒ 路径可达（可行性初筛通过） |

### HyperOS 框架层加固（提权后依然受限）

在 `u:r:su:s0` 域下（`uid=0`，`CapEff=000001ffffffffff`）：

```
am force-stop ...        → Failure calling service activity:  Failed transaction (2147483646)
pm list packages         → Failure calling service package:   Failed transaction (2147483646)
cmd appops ...           → Failure calling service appops:    Failed transaction (2147483646)
settings get/put         → Failure calling service settings:  Failed transaction (2147483646)
```

降权到 `su 2000` 也一样被拦（context 仍是 `u:r:su:s0`）。

**⇒ 提权后不能使用框架接口。** 可行的手段只剩：**内核 / sysfs 节点** + **直接按 PID 杀进程**。

`dumpsys` 反而可用（能读），例如取前台包名：

```sh
dumpsys window | grep -m1 'mCurrentFocus' | sed -n 's/.*u0 \([^ /]*\)\/.*/\1/p'
```

## 内核提供的可调 / 可写节点盘点

| 节点 | 权限 | 可写 |
|---|---|---|
| `/proc/sys/walt/sched_fmax_cap` | `-rw-r--r--` | ✅ 按簇设频率天花板 |
| `/sys/devices/system/cpu/cpufreq/policyN/scaling_max_freq` | `-rw-rw-r--` | ✅ |
| `/sys/devices/system/cpu/cpufreq/policyN/walt/target_load_thresh` | **`-r--r--r--`** | ❌ root 也写不进 |
| `/sys/devices/system/cpu/cpufreq/policyN/walt/target_loads` | **`-r--r--r--`** | ❌ |
| `/sys/devices/system/cpu/cpufreq/policyN/walt/hispeed_freq` | **`-r--r--r--`** | ❌ |
| `/sys/devices/system/cpu/cpufreq/policyN/walt/pl`、`rtg_boost_freq` | `-rw-r--r--` | ✅ |
| `/sys/class/kgsl/kgsl-3d0/max_gpuclk` | 可写 | ✅ |
| `/sys/class/kgsl/kgsl-3d0/devfreq/max_freq` | **`-r--r--r--`** | ❌ |
| `/sys/module/migt/parameters/{flt_target_fps,glk_maxfreq,migt_freq,migt_ceiling_freq,fps_variance_ratio}` | `0666`/`0644` | ✅ |
| `/sys/module/migt/parameters/glk_disable` | **`-r--r--r--`** | ❌ |
| `/data/system/mcd/*`（`policy`/`raf_debug`/`fi`/`migl`…） | `0777` | ✅ 任务放置控制 |
| `/sys/block/zram0/comp_algorithm` | 存在但切不动 | ❌（当前 `lzo-rle`，本就合理） |

**要点**：小米云控（`com.xiaomi.joyose`）写的正是那些 `0444` 只读节点 —— 它以 `system` 用户身份写。**任何第三方"性能模块"对它们的写入都是静默失败的。** 详见 [`08-tuning-annex.md`](08-tuning-annex.md)。
