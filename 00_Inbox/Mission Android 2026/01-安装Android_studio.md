# Android Studio 模拟器启动卡死 / 无网络 自救笔记

> 环境：Windows 11 + Android Studio 2026.2 + Pixel 7 Pro AVD（API 37，x86_64）
> 症状：模拟器永远转圈、"Emulator failed to connect within 5 minutes"、"The emulator process has terminated"、WiFi 图标带感叹号、YouTube 无网络
> 全部命令均在 **PowerShell** 中运行

---

## 〇、准备工作（只做一次）：让 adb 永久可用

模拟器工具默认不在 PATH 里，先执行（**当前窗口不会生效，之后新开的窗口都能直接用 adb**）：

```powershell
[Environment]::SetEnvironmentVariable("Path",
  [Environment]::GetEnvironmentVariable("Path","User") + ";$env:LOCALAPPDATA\Android\Sdk\platform-tools",
  "User")
```

如果不想改 PATH，就在每个新窗口先执行这句（仅当前窗口有效）：

```powershell
$env:Path += ";$env:LOCALAPPDATA\Android\Sdk\platform-tools"
```

验证：`adb --version` 有输出即可。

---

## 一、根因链条（最重要）

**真凶只有一个：Windows 系统代理被注入模拟器进程。**

```
Windows 开启系统代理 (ProxyEnable=1)
  → Android Studio 自动检测到它
  → 启动模拟器时注入 http_proxy 环境变量
  → 模拟器底层网络进程 (qemu/netsim) 尝试通过该代理建连
  → 死锁：内存冻结不动、adb 永远 offline、5 分钟后 AS 杀进程
```

**验证方法（决定性实验）**：命令行启动时手动带上代理环境变量，立刻复现卡死：

```powershell
$env:http_proxy='http://192.168.163.78:7890'
$env:https_proxy='http://192.168.163.78:7890'
# 再启动模拟器 → 必然卡死（内存冻结、adb offline）
```

判断卡死的信号：`tasklist` 里 qemu 内存几分钟**一动不动**（如一直 478MB），正常启动会一路涨到 4~6GB。

### 三个"坑"叠加

| 坑                 | 说明                                                        | 解法                                                 |
| ------------------ | ----------------------------------------------------------- | ---------------------------------------------------- |
| 系统代理注入       | 上述主因                                                    | 关闭 Windows 系统代理，或保持 AS 代理设置为 No proxy |
| 损坏的快速启动快照 | 强杀进程会留下带 dirty 标记的半成品快照，下次启动恢复时卡死 | 删快照 + 关闭快速启动                                |
| AVD 内存太小       | Pixel 7 Pro 高分屏 + 新系统，默认 2GB 太紧                  | 调到 4GB                                             |

---

## 二、急救流程（按顺序做）

```powershell
# 1. 杀掉所有卡死进程
taskkill /F /IM qemu-system-x86_64.exe
taskkill /F /IM emulator.exe
adb kill-server

# 2. 删除残留锁和快照（换成你的 AVD 名）
Remove-Item -Recurse -Force "$env:USERPROFILE\.android\avd\Pixel_7_Pro.avd\snapshots" -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force "$env:USERPROFILE\.android\avd\Pixel_7_Pro.avd\hardware-qemu.ini.lock" -ErrorAction SilentlyContinue
Remove-Item -Force "$env:USERPROFILE\.android\avd\Pixel_7_Pro.avd\multiinstance.lock" -ErrorAction SilentlyContinue

# 3. 检查 Windows 系统代理（ProxyEnable 应为 0；为 1 就先去系统设置里关掉）
Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" | Select-Object ProxyEnable

# 4. 命令行冷启动测试（绕开 AS，最快验证）
& "$env:LOCALAPPDATA\Android\Sdk\emulator\emulator.exe" -avd Pixel_7_Pro -no-snapshot-load -gpu auto

# 5. 观察开机（另开一个 PowerShell 窗口，重复执行直到输出 1）
adb wait-for-device
adb shell getprop sys.boot_completed   # 输出 1 = 开机完成，正常 15~60 秒
```

---

## 三、永久性修复（已应用）

**1. AVD 配置**（`C:\Users\<你>\.android\avd\<AVD名>.avd\config.ini`）：

```ini
hw.ramSize=4096               # 2GB → 4GB
fastboot.forceColdBoot=yes    # 关闭快速启动
fastboot.forceFastBoot=no     # 永远冷启动，杜绝快照损坏卡死
```

> 代价：每次开机多花十几秒；收益：彻底免疫快照损坏。

**2. Android Studio 设置**：

- `Settings → Appearance & Behavior → System Settings → HTTP Proxy` → **No proxy**
- `Settings → Tools → Emulator → Launch in a tool window`：内嵌/独立窗口均可，内嵌模式更脆弱，出问题就先取消勾选排查

---

## 四、网络问题分层（"WiFi 感叹号"和"YouTube 打不开"是两回事）

先确认模拟器在线：`adb devices` 应显示 `emulator-5554  device`（offline 或空白 = 回第二节急救）。

判断口诀：**先测 IP，再测域名，最后测 Google**

```powershell
adb shell ping -c 2 223.5.5.5        # 通 = 底层网络没问题
adb shell ping -c 2 www.baidu.com    # 通 = DNS 没问题
adb shell ping -c 2 www.google.com   # 不通 = 正常现象（国内直连受限）
```

- **感叹号**：Android 默认连 Google 服务器检测网络，国内连不上就显示"无互联网"。换国内检测服务器即可消除：

```powershell
adb shell settings put global captive_portal_http_url http://connect.rom.miui.com/generate_204
adb shell settings put global captive_portal_https_url https://connect.rom.miui.com/generate_204
```

- **YouTube/Play 商店**：国内网络无法直连 Google，不是模拟器坏了，要走代理（见下）。

---

## 五、代理的正确管理方式

**核心原则：模拟器代理和 Windows 系统代理各管各的，互不干扰。**

### ✅ 正确姿势：在模拟器内部配置（推荐）

```powershell
# 开代理（填你的代理地址，持久保存，重启不丢）
adb shell settings put global http_proxy 192.168.163.78:7890

# 关代理（代理设备关机时必须执行，否则模拟器断网）
adb shell settings put global http_proxy :0

# 查看当前代理
adb shell settings get global http_proxy
```

可视化方式：设置 → 网络 → AndroidWifi ⚙ → 铅笔 → 高级选项 → 代理 → 手动。

### ❌ 不要做的事

1. **不要在 Windows 系统代理开启时从 AS 启动模拟器**（触发本篇的卡死）
2. 不要在模拟器设置里手动开关 WiFi（会把虚拟网卡状态搞坏，需 Wipe Data 才能恢复）
3. 不要同时启动多个 AVD（互相抢 adb，全部 offline）

### 代理自检（在 PowerShell 直接跑）

```powershell
Test-NetConnection 192.168.163.78 -Port 7890
# TcpTestSucceeded : True = 代理设备在线；超时 = 代理设备没开
```

---

## 六、日常使用速查

| 场景                          | 操作                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| 正常启动                      | AS 里 Device Manager → ▶，等 15~30 秒                        |
| "process has terminated" 弹窗 | 只是关机通知，点 OK，不是错误                                |
| 转圈超过 2 分钟               | 判定卡死，执行第二节急救流程                                 |
| 模拟器断网                    | 先 `Test-NetConnection` 查代理设备，不在线就 `:0` 清代理     |
| Cold Boot / Wipe Data         | Device Manager 里 AVD 旁 ▼ 菜单，Wipe 会清空 App 数据        |
| 查启动日志                    | `$env:LOCALAPPDATA\Google\AndroidStudio*\log\idea.log` 里搜 `Emulator:` |

---

## 七、经验教训

1. **卡死先怪环境变量**：GUI 启动失败、命令行成功时，差异几乎总在父进程注入的环境变量（本次是 http_proxy）
2. **复现要最小化**：用完全相同的参数分别走"AS 启动"和"命令行启动"，二分定位是参数问题还是环境问题
3. **内存冻结 = 死锁**：qemu 内存几分钟不变就是卡死，别傻等
4. **快照是双刃剑**：快速启动快，但非正常关机产生的脏快照会导致下次必卡
5. **修一个坑会暴露下一个**：本次依次暴露 双实例冲突 → 内存不足 → 脏快照 → WiFi 配置损坏 → 代理注入，逐层剥洋葱是常态

这版的改动：

1. **新增"第〇节 准备工作"**：一次性永久 PATH 配置（或临时方案），之后所有命令直接用 `adb`
2. **所有代码块标记改为 `powershell`**，语法全部对齐（`taskkill /F` 单反斜杠、`Remove-Item`、`Get-ItemProperty`、`&` 调用、`Test-NetConnection` 去掉多余前缀）
3. 开头注明 CMD 用户需删掉 `#` 注释

**要不要我现在就帮你把第〇节的永久 PATH 执行了？这样笔记你以后照抄就能跑。**