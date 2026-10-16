# 02 · 临时 root 完整配方（已验证可用）

**这不是漏洞利用，是厂商遗留的注入通道** —— 所以它稳定、可复现、不丢数据。
代价是：**重启即失效**。

## 配方

```bash
# ① 进入 bootloader
adb reboot bootloader

# ② 打开 permissive 并附加参数
fastboot oem set-gpu-preemption 0 androidboot.selinux=permissive

# ③ 继续启动
fastboot continue

# ④ 等系统起来
adb wait-for-device
while [ "$(adb shell getprop sys.boot_completed | tr -d '\r')" != "1" ]; do sleep 3; done

# ⑤ 立刻注入（趁 Permissive 窗口）
adb push ksud /data/local/tmp/ksud
adb shell chmod 777 /data/local/tmp/ksud
adb shell "service call miui.mqsas.IMQSNative 21 \
  i32 1 s16 '/data/local/tmp/ksud' \
  i32 1 s16 'late-load' \
  s16 '/data/local/tmp/ksulog.txt' i32 120"
```

**结果**：注入进程获得

```
uid=0(root) gid=1000(system) context=u:r:hypsys_ssi_default:s0
```

## 关键约束（踩过的坑）

### 1. 注入只在 SELinux Permissive 期间有效

- `ro.boot.selinux` 读出来**恒为 `enforcing`**，即便传了 fastboot 参数。**用 `getenforce` 判断。**
- 注入的 `late-load` 会触发模块 `service.sh`，而模块脚本常会把 SELinux **翻回 Enforcing**。
  例如 `h_reboot/service.sh` 里有：

  ```sh
  if [ "$(getenforce)" = "Permissive" ]; then setenforce 1; fi
  ```

  ⇒ 一旦翻回去，后续注入就失败。**所以必须在窗口内一次成功。**
- **Permissive 窗口实测：启动后稳定 ≥ 50 秒。** 足够。

### 2. 组件信息

| 项 | 值 |
|---|---|
| `ksud` 版本 | `3.1.0-29-gf0615d3c` |
| `ksud` MD5 | `06FE421B10A7BB433B8C6A52F8B20C6D` |
| 大小 | 3,935,760 B |

### 3. `ksud late-load` 的两条失败模式

| 现象 | 原因 |
|---|---|
| `Cannot parse kallsyms / Permission denied (os error 13)` | 以 **shell 身份**跑 `late-load` ⇒ 必须走 mqsas 注入 |
| `exit=0` 但 `ksud debug version` → `Kernel Version: 0`，`/system/bin/su` 不存在 | 多半是**没有随附 `kernelsu.ko`**（装的是官方 KernelSU 管理器，不是自带 `libksud.so` 的 ReSukiSU 分支） |

### 4. 注入进度的观察点

```sh
# 注入后轮询
adb shell "su -c id"
```

### 5. 恢复路径

实测 **100% 可靠**：重新走一遍 ①→⑤ 即可。**不会丢数据。**

## 工程化封装

已经做成可复用脚本（Windows 主机侧）：

```
C:\rootkit\run-root.cmd          ← 双击即用（桌面有快捷方式 一键Root.lnk）
C:\rootkit\root.ps1              ← 主逻辑
C:\rootkit\root-log.txt          ← 日志
```

`root.ps1` 的行为：

1. 检查素材（`adb.exe` / `fastboot.exe` / `ksud`）
2. **若已是 root，直接跳过注入**，只重新应用调优
3. 自动 `adb kill-server` / `start-server` 重连
4. 自动判断当前在 adb 还是 fastboot（需要时 `reboot bootloader`）
5. `oem set-gpu-preemption 0 androidboot.selinux=permissive` → `continue`
6. **零延迟**推送 + 注入（抢 Permissive 窗口）
7. 轮询 `su -c id`，最多 90 秒
8. 失败自动重试，最多 3 轮

### Windows 脚本的两个致命编码坑（务必遵守）

| 坑 | 现象 | 解法 |
|---|---|---|
| PowerShell `-File` 把**无 BOM 的 `.ps1`** 当 ANSI/GBK 读 | 中文乱码 → 语法错误（`Unexpected token ')'`、`Missing closing '}'`） | `.ps1` 必须写 **UTF-8 *带* BOM**（`ef bb bf`） |
| `cmd.exe` 把 **UTF-8 `.cmd` 里的中文路径**当 GBK 读 | `set D=...` 被截断成 `"\adb.exe"` | **脚本里所有路径用纯 ASCII**（例如 `C:\rootkit\`） |

### 另一个坑：PowerShell 吞变量

```powershell
# ❌ $(...) 和 $c 会被 PowerShell 吃掉，变成 adb 去执行 id
adb shell "su -c 'for p in /proc/*; do echo $p; done'"
```

**解法**：在本地写 `.sh` → `adb push` → `adb shell "su -c 'sh /path/x.sh'"`。

## 这条路的价值

它本身不持久，但它说明了一件事：

> **厂商必须留一条"不进 fastboot 也能把权限起来"的路**（这里是 `mqsas` 服务）。
>
> 顺着"厂商自己留的可用通道"去找，往往比硬啃内核漏洞更接近持久化。
> 值得继续枚举：还有哪些 MIUI/HyperOS 系统服务能被以类似方式调用？它们的权限边界在哪？能否在开机流程里自动触发？
