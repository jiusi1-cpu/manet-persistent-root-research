# 08 · 附：性能/续航调优实测（同机副产品）

做持久 root 研究时顺手做的调优。**三个反直觉的实测发现**，对同机型 / 同 SoC 的人有参考价值。

数据全部来自同一台设备（`manet` / HyperOS 3.0.9.0），非资料转述。

---

## 🔴 发现 1：电池是硬天花板，软件补不回来

```
charge_full_design = 5000000 uAh   （设计 5000 mAh）
charge_full        = 3320000 uAh   （实际 3320 mAh）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
健康度 = 66%          循环次数 = 1751
```

`dumpsys batterystats` 印证：

```
Estimated battery capacity: 5000 mAh
Last/Min/Max learned battery capacity: 3320 mAh
```

**⇒ 丢了三分之一。任何软件调优只能省"软件开销"那部分，补不回容量损失。**

---

## 🔴 发现 2：耗电第一来源是内存，不是温控

### 温度实测：完全没热

```
cpu-0-0-0  = 38.2 °C
gpuss-*    = 34.9 ~ 35.6 °C
ddr        = 35.9 °C
kgsl temp  = 36.4 °C
throttling = 0          ← 完全未触发
```

### 但内存被榨干

```
Mem:  15160 MB 总量,  只用剩 417 MB 空闲
Swap: 16383 MB,       已用 6233 MB
pswpin   = 8353449 页
pswpout  = 15312064 页     ← 累计换出 ≈ 61 GB
```

### 实测放电分解（一次完整放电，3h21m）

```
总放电 1346 mAh
├─ cpu          629 mAh   ← 占 48%，单项最大
├─ screen       412 mAh
├─ mobile_radio 112 mAh
├─ bluetooth     15.6 mAh
├─ video          8.19 mAh
├─ audio          7.66 mAh
├─ wakelock       4.61 mAh（34m32s）
├─ camera         2.94 mAh
├─ phone          1.73 mAh
└─ sensors        1.38 mAh
```

**⇒ CPU 吃掉近一半电，而机器一点都不热 —— 说明是 `zram` 反复压缩/换页烧的 CPU，不是频率跑太高。**

### 处理后的效果

收敛后台常驻后：

```
Mem free:  417 MB  →  4 GB+
swap 换出：基本停止
```

---

## 🔴 发现 3：小米云控控制的那些节点，root 也写不进

从 `com.xiaomi.joyose` 的 `databases/teg_config.db` 里解出云控配置，
确认它下发的正是这些 sysfs 节点（`permission: "root"` + `cmd` 形式）：

```
/sys/devices/system/cpu/cpufreq/policy{0,2,5,7}/walt/target_load_thresh
/sys/devices/system/cpu/cpufreq/policy{0,2,5,7}/walt/target_loads
/sys/devices/system/cpu/cpu7/core_ctl/max_cpus
/sys/devices/system/cpu/cpu7/core_ctl/min_cpus
/sys/module/migt/parameters/*
/data/system/mcd/policy          ← 任务放置
```

**但它们的权限是 `-r--r--r--`（0444）：**

| 节点 | 权限 |
|---|---|
| `walt/target_load_thresh` | **`-r--r--r--`** |
| `walt/target_loads` | **`-r--r--r--`** |
| `walt/hispeed_freq` | **`-r--r--r--`** |
| `walt/pl`、`rtg_boost_freq` | `-rw-r--r--` ✅ |

即便 `uid=0`、`CapEff=000001ffffffffff`，写 `0444` 的 sysfs 也是 `Permission denied`。

**⇒ 网上流传的"性能模块"里对 `target_load_thresh` 之类的写入，绝大多数是静默失败的。**
只有小米的 `system` 用户（通过 perfmgr / `mcd` 通道）能写。

---

## 真正可零成本生效的旋钮

| 节点 | 权限 | 用途 |
|---|---|---|
| `/proc/sys/walt/sched_fmax_cap` | `-rw-r--r--` | **按簇设频率天花板**（4 个值，空格分隔） |
| `/sys/.../policyN/scaling_max_freq` | `-rw-rw-r--` | 单簇上限 |
| `/sys/class/kgsl/kgsl-3d0/max_gpuclk` | 可写 | **GPU 封顶**（`devfreq/max_freq` 反而写不进） |
| `/sys/module/migt/parameters/migt_freq` | `0666` | 逐核频率上限 |
| `/sys/module/migt/parameters/migt_ceiling_freq` | `0666` | 逐核天花板 |
| `/sys/module/migt/parameters/flt_target_fps` | `0644` | 帧目标（**语义未验证，慎用**） |
| `/sys/module/migt/parameters/glk_maxfreq`、`glk_minfreq` | `0666` | 小米帧限速器 |
| `/data/system/mcd/*` | `0777` | 任务放置 |

**实测可稳住的值**（空闲/负载下均不回压）：

```
sched_fmax_cap = 2265600 2400000 2300000 2745600   # 原 2265600 3148800 2956800 3302400
max_gpuclk     = 578000000                          # 原 903000000
```

---

## ⚠️ 提权后**不能**用框架接口

```
am / pm / cmd appops / settings  →  Failed transaction (2147483646)
```

在 `u:r:su:s0` 域全被 HyperOS 拦截（降权到 `su 2000` 也一样）。
**所以：禁用包、强制停止、改设置这些常规手段全部不可用。**

能用的只有：

- 内核 / sysfs 节点
- **直接按 PID 杀进程**（`/proc/*/cmdline` 匹配 + `kill -9`）
- `dumpsys`（只读，可用）

### 一个实用技巧

能从 su 域读到前台包名，用来给"后台收敛"加保护，避免误杀正在用的 App：

```sh
dumpsys window | grep -m1 'mCurrentFocus' | sed -n 's/.*u0 \([^ /]*\)\/.*/\1/p'
# → com.ss.android.ugc.aweme
```

### 另一个实测教训：杀了又起，是净亏损

无节制地反复杀 `com.google.android.gms`，会导致它**反复冷启动**：

```
实测：3 个 GMS 进程合计占 14% 单核（top: 57.1 + 35.7 + 28.5）
```

⇒ 加了 **300 秒冷却退避**后降到 ~1.4%。
**杀后台要算"重启成本"，不是杀得越勤越好。**

---

## 其它实测边界

| 项 | 结果 |
|---|---|
| `zram0` 压缩算法 | `lzo `[`lzo-rle`]` lz4 zstd`；**切不动**（写 `comp_algorithm` 失败）。`lzo-rle` 本身是平衡解，不必改 |
| `glk_disable` | `-r--r--r--`，**写不进**（默认已是 `1`，即限速器关闭） |
| 屏幕模式 | `mActiveModeId=2` = **120 Hz 已激活**；支持 60 / 90 / 120 三档 |
| `walt/target_load_thresh` 当前值 | 1024（= 关闭该限制的哨兵值） |

---

## 一句话总结

**续航差的根因是「电池 66% 老化 + 内存被榨干导致 zram 换页烧 CPU」，不是温控、也不是频率跑太高。**
把后台常驻收敛掉、给 CPU/GPU 压个天花板，比折腾那些写不进去的"调频模块"有效得多。
