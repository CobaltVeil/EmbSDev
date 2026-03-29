# Nordic nRF Connect SDK Windows 开发环境搭建指南

> **适用芯片**：nRF52840 / nRF5340 / nRF54L15 系列  
> **操作系统**：Windows 10/11 64位  
> **SDK 版本**：v2.9.2  
> **记录日期**：2026-03-29  
> **文档性质**：个人安装实录 + 踩坑总结

---

## 目录

1. [前提条件](#1-前提条件)
2. [安装 Git](#2-安装-git)
3. [安装 nRF Command Line Tools + J-Link](#3-安装-nrf-command-line-tools--j-link)
4. [安装 VS Code 扩展](#4-安装-vs-code-扩展)
5. [安装 nrfutil](#5-安装-nrfutil)
6. [安装 Toolchain（命令行方式）](#6-安装-toolchain命令行方式)
7. [安装 SDK（west 方式）](#7-安装-sdkwest-方式)
8. [在 VS Code 中关联 SDK](#8-在-vs-code-中关联-sdk)
9. [创建第一个项目并编译](#9-创建第一个项目并编译)
10. [目录结构总览](#10-目录结构总览)
11. [关键踩坑记录](#11-关键踩坑记录)
12. [常用命令速查](#12-常用命令速查)

---

## 1. 前提条件

开始之前，确认以下软件已安装：

| 软件 | 版本 | 说明 |
|---|---|---|
| Windows | 10/11 64位 | 操作系统 |
| VS Code | 最新版 | 主 IDE |
| Python | 3.12 | 系统 Python（不影响 SDK 自带 Python） |

> ⚠️ **路径铁律**：所有安装路径不能包含**中文、空格、括号**等特殊字符，建议统一安装在 `D:\` 根目录下。

---

## 2. 安装 Git

**下载地址**：https://git-scm.com/download/win

**安装选项**：
- ✅ 勾选 "Add Git to PATH"
- 换行符选项保持默认即可

**默认安装路径**：
```
C:\Program Files\Git\
```

**验证安装**：
```powershell
git -v
# 预期输出：git version 2.53.0.windows.2
```

> ⚠️ **踩坑**：安装完成后检查 Git 全局代理配置，如果之前安装过 GitHub 加速工具（如 FastGithub、Steam++），可能会残留 `url.insteadOf` 配置干扰后续 `west update`，需要提前清理：
> ```powershell
> git config --global --list | findstr url
> # 如果看到 ghp.ci 或 ghproxy 相关配置，执行：
> git config --global --unset-all url.https://ghp.ci/https://github.com/.insteadof
> git config --global --unset-all url.https://mirror.ghproxy.com/https://github.com/.insteadof
> ```

---

## 3. 安装 nRF Command Line Tools + J-Link

### 3.1 安装 SEGGER J-Link（必须先装）

**下载地址**：https://www.segger.com/downloads/jlink/

选择 **J-Link Software and Documentation Pack for Windows 64-bit**。

**默认安装路径**：
```
C:\Program Files\SEGGER\JLink\
```

关键文件确认：
```
C:\Program Files\SEGGER\JLink\JLinkARM.dll  ← 必须存在
```

### 3.2 安装 nRF Command Line Tools

**下载地址**：https://www.nordicsemi.com/Products/Development-tools/nRF-Command-Line-Tools/Download

选择 **Windows x64** 版本。

**默认安装路径**：
```
C:\Program Files\Nordic Semiconductor\nrf-command-line-tools\
```

### 3.3 验证安装

```powershell
nrfjprog --version
```

**正常输出**（未连接开发板时）：
```
# 未连接开发板时会有 error -256 提示，这是正常现象
# 只要 --help 能正常显示帮助文本，说明安装成功
```

> ⚠️ **踩坑**：`nrfjprog --version` 报 `Could not find a JLinkARM.dll` → J-Link 未安装或路径注册表错误。  
> 报 `error -256` → J-Link 已找到，但没有连接开发板，**属于正常现象**，不是错误。

---

## 4. 安装 VS Code 扩展

在 VS Code 中按 `Ctrl+Shift+X`，搜索并安装：

```
nRF Connect for VS Code Extension Pack
```

安装完成后左侧活动栏出现 Nordic **N 图标**。

**VS Code settings.json 配置**（`Ctrl+Shift+P` → `Open User Settings (JSON)`）：

```json
{
    "nrf-connect.enableTelemetry": true,
    "nrf-connect.downloadMirror": "china",
    "nrf-connect.toolchainManager.installDirectory": "D:\\ncs"
}
```

> ⚠️ **踩坑**：`toolchainManager.installDirectory` 只能指定**根目录**（如 `D:\\ncs`），不能指定到具体版本子文件夹。  
> `toolchainManager.indexURL` 不能填本地路径，否则版本列表为空。

---

## 5. 安装 nrfutil

nrfutil 是 Nordic 新一代命令行工具，用于管理 Toolchain 和 SDK，**比 VS Code 图形界面下载更稳定，支持断点续传**。

**下载地址**：https://www.nordicsemi.com/Products/Development-tools/nRF-Util/Download

下载 Windows x64 的 `nrfutil.exe`。

**安装位置**：
```
D:\nrfutil\nrfutil.exe
```

**添加到系统 PATH**：
- Win + S → 搜索"编辑系统环境变量" → 环境变量 → 系统变量 → Path → 新建 → 输入 `D:\nrfutil`

**验证安装**：
```powershell
nrfutil --version
# 预期输出：nrfutil 8.1.1
```

**安装 toolchain-manager 插件**：
```powershell
nrfutil install toolchain-manager
nrfutil toolchain-manager --version
# 预期输出：nrfutil-toolchain-manager 0.15.0
```

---

## 6. 安装 Toolchain（命令行方式）

### 6.1 查看可用版本

```powershell
nrfutil toolchain-manager search
```

### 6.2 获取下载直链（用于浏览器下载）

由于国内网络对 Nordic 服务器的 TLS 兼容性问题，命令行直接下载可能失败。推荐**先获取直链，再用浏览器下载**：

```powershell
nrfutil toolchain-manager install --ncs-version v2.9.2 --install-dir D:\ncs --log-level trace 2>&1 | Select-String -Pattern "https://"
```

输出示例：
```
https://files.nordicsemi.com/NCS/external/bundles/v3/ncs-toolchain-x86_64-windows-b620d30767.tar.gz
```

将这个链接复制到浏览器（确保 VPN 全局模式开启）下载，文件约 **1~2 GB**。

### 6.3 从本地文件安装

```powershell
nrfutil toolchain-manager install --install-dir D:\ncs --toolchain-bundle "D:\UserFiles\Downloads\ncs-toolchain-x86_64-windows-b620d30767.tar.gz"
```

**安装路径**：
```
D:\ncs\toolchains\b620d30767\    ← Toolchain 实际安装位置
```

**验证安装**：
```powershell
nrfutil toolchain-manager list --install-dir D:\ncs
# 预期输出：
# Version                                    Toolchain
# 7a8f192072e5d8706b5453cdacf88f30431b3416  D:\ncs\toolchains\b620d30767
```

---

## 7. 安装 SDK（west 方式）

### 7.1 进入 Toolchain Shell 环境

```powershell
nrfutil toolchain-manager launch --terminal --install-dir D:\ncs
```

终端提示符变为 `(b620d30767) C:\Windows\System32>` 说明成功进入。

### 7.2 配置 Git 代理（国内网络必须）

```cmd
# 清除旧代理
git config --global --unset http.proxy
git config --global --unset https.proxy

# 设置正确代理（端口根据 VPN 软件实际端口填写）
# v2rayN 默认端口为 10808
git config --global http.proxy http://127.0.0.1:10808
git config --global https.proxy http://127.0.0.1:10808

# 关闭 SSL 验证（解决 Hysteria2 协议的 TLS 兼容问题）
git config --global http.sslVerify false

# 清除 GitHub 加速镜像干扰（如有）
git config --global --unset-all url.https://ghp.ci/https://github.com/.insteadof
git config --global --unset-all url.https://mirror.ghproxy.com/https://github.com/.insteadof
```

### 7.3 初始化 west 工作区

```cmd
west init -m https://github.com/nrfconnect/sdk-nrf --mr v2.9.2 D:\ncs\v2.9.2
```

**预期输出**：
```
=== Initialized. Now run "west update" inside D:\ncs\v2.9.2.
```

### 7.4 拉取全部 SDK 源码

```cmd
D:
cd D:\ncs\v2.9.2
west update
```

> ⚠️ 此步骤会从 GitHub 拉取约 **3~5 GB** 数据，包含 Zephyr、nrfxlib、MCUboot、Matter 等所有依赖仓库，耗时 **30~60 分钟**，请保持网络畅通。

### 7.5 注册 Zephyr CMake 包

```cmd
west zephyr-export
```

**预期输出**：
```
Zephyr (D:/ncs/v2.9.2/zephyr/share/zephyr-package/cmake)
has been added to the user package registry in:
HKEY_CURRENT_USER\Software\Kitware\CMake\Packages\Zephyr
```

### 7.6 安装 Python 依赖

```cmd
pip install -r D:\ncs\v2.9.2\zephyr\scripts\requirements.txt
```

所有包显示 `Requirement already satisfied` 即为成功（Toolchain 自带 Python 环境已包含所有依赖）。

---

## 8. 在 VS Code 中关联 SDK

1. 打开 VS Code，按 `Ctrl+Shift+P` → `Developer: Reload Window`
2. 点击左侧 **N 图标** → **Manage SDKs** → **Manage west workspace...**
3. 点击 **"Open west manifest"**
4. 选择 `nrf  d:\ncs\v2.9.2\nrf`（会自动识别）

关联成功后，左侧面板 APPLICATIONS 区域激活，可以创建和管理项目。

---

## 9. 创建第一个项目并编译

### 9.1 创建 hello_world 示例

1. 点击 **"Create a new application"**
2. 选择 **"Copy a sample"**
3. 搜索并选择 `hello_world`
4. 项目路径填写：`D:\projects\hello_world`
5. 回车确认，点击 **"Open"** 打开

### 9.2 添加编译配置

在 APPLICATIONS 面板里，点击 `hello_world` 旁的 **"Add Build Configuration"**，根据手中的开发板选择目标：

| 开发板 | Board Target |
|---|---|
| nRF52840 DK | `nrf52840dk/nrf52840` |
| nRF5340 DK | `nrf5340dk/nrf5340/cpuapp` |
| nRF54L15 DK | `nrf54l15dk/nrf54l15/cpuapp` |

点击 **Build** 开始编译。

### 9.3 编译成功标志

```
[100%] Linking C executable zephyr/zephyr.elf
Memory region         Used Size  Region Size  %age Used
           FLASH:       26356 B       1 MB      2.51%
             RAM:        5144 B     256 KB      1.96%
[100%] Built target zephyr/zephyr.hex
```

输出固件位置：
```
D:\projects\hello_world\build\zephyr\zephyr.hex
```

### 9.4 烧录并查看串口输出

连接开发板后点击 **Flash**，烧录成功后点击 **Open terminal** 查看串口输出：
```
*** Booting nRF Connect SDK v2.9.2 ***
Hello World! nrf52840dk/nrf52840
```

---

## 10. 目录结构总览

```
D:\
├── ncs\                              ← SDK 根目录
│   ├── toolchains\
│   │   └── b620d30767\              ← Toolchain（GCC、CMake、Ninja、west、Python）
│   └── v2.9.2\                      ← SDK 源码
│       ├── .west\                   ← west 工作区配置
│       ├── zephyr\                  ← Zephyr RTOS 内核
│       ├── nrf\                     ← Nordic 专有驱动和示例
│       ├── nrfxlib\                 ← BLE 协议栈等无线库
│       ├── bootloader\mcuboot\      ← MCUboot 安全引导
│       └── modules\                 ← 第三方模块（mbedTLS、Matter等）
├── nrfutil\
│   └── nrfutil.exe                  ← nrfutil 工具
├── projects\
│   └── hello_world\                 ← 你的应用项目（建议放这里）
└── UserFiles\Downloads\
    └── ncs-toolchain-x86_64-windows-b620d30767.tar.gz  ← Toolchain 原始包（可保留备用）

C:\
├── Program Files\SEGGER\JLink\
│   └── JLinkARM.dll                 ← J-Link 核心驱动
└── Program Files\Nordic Semiconductor\nrf-command-line-tools\
    └── nrfjprog.exe                 ← nrfjprog 烧录工具
```

---

## 11. 关键踩坑记录

### ❌ 坑1：nrfjprog 报 `Could not find a JLinkARM.dll`
**原因**：J-Link 驱动未安装或安装顺序错误。  
**解决**：先安装 J-Link，再安装 nRF Command Line Tools。

### ❌ 坑2：nrfjprog 报 `error -256`
**原因**：误以为是错误，实际上是**正常现象**。  
**解决**：这是因为没有连接开发板，工具本身安装正确。

### ❌ 坑3：VS Code Toolchain 版本列表为空
**原因**：`settings.json` 中 `toolchainManager.indexURL` 被错误地填为本地路径。  
**解决**：删除该配置项，只保留 `installDirectory` 和 `downloadMirror`。

### ❌ 坑4：nrfutil 下载 Toolchain 速度极慢（0.1 Mbps）
**原因**：即使设置了 `china` 镜像，国内访问 Nordic 服务器速度仍不稳定。  
**解决**：用 `--log-level trace` 获取直链，改用浏览器（走 VPN）下载。

### ❌ 坑5：nrfutil 设置代理后报 `TLS close_notify` 错误
**原因**：nrfutil 使用的 Rust TLS 实现与 Hysteria2 协议不兼容。  
**解决**：用浏览器直接下载 `.tar.gz` 包，再用 `--toolchain-bundle` 参数本地安装。

### ❌ 坑6：`west init` 报 `ghp.ci` 域名无法连接
**原因**：系统 Git 全局配置中有残留的 GitHub 加速镜像 `url.insteadOf` 配置。  
**解决**：
```cmd
git config --global --unset-all url.https://ghp.ci/https://github.com/.insteadof
git config --global --unset-all url.https://mirror.ghproxy.com/https://github.com/.insteadof
```

### ❌ 坑7：`west update` 命令报 "no west workspace found"
**原因**：在 Windows 中 `cd D:\ncs\v2.9.2` 不会自动切换盘符。  
**解决**：必须先输入 `D:` 切换盘符，再 `cd D:\ncs\v2.9.2`。

### ❌ 坑8：Git SSL 握手失败
**原因**：Hysteria2 协议的代理与 Git 的 schannel TLS 引擎不兼容。  
**解决**：
```cmd
git config --global http.sslVerify false
```

---

## 12. 常用命令速查

### 进入 Toolchain Shell
```powershell
nrfutil toolchain-manager launch --terminal --install-dir D:\ncs
```

### 查看已安装 Toolchain
```powershell
nrfutil toolchain-manager list --install-dir D:\ncs
```

### 查看可用 SDK 版本
```powershell
nrfutil toolchain-manager search
```

### 更新 SDK（进入 Toolchain Shell 后执行）
```cmd
cd D:\ncs\v2.9.2
west update
```

### 检查开发板连接
```powershell
nrfjprog --ids
```

### 烧录固件
```powershell
nrfjprog --program D:\projects\hello_world\build\zephyr\zephyr.hex --chiperase --reset -f NRF52
```

---

## 参考资料

| 资源 | 地址 |
|---|---|
| Nordic 官方文档 | https://docs.nordicsemi.com |
| Nordic 开发者学院（免费课程） | https://academy.nordicsemi.com |
| 开发者社区问答 | https://devzone.nordicsemi.com |
| nRF Connect for VS Code 文档 | https://nrfconnect.github.io/vscode-nrf-connect |
| Zephyr 官方文档 | https://docs.zephyrproject.org |

---

*文档版本：v1.0 | 基于 NCS v2.9.2 + Windows 11 实际安装记录*
