![AOSP Dual-OS Banner](docs/assets/banner.jpg)

# AOSP Dual-OS (Android + GNU/Linux) 🚀

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![AOSP Version](https://img.shields.io/badge/AOSP-Android_15%2F16-green.svg)](https://source.android.com)
[![Architecture](https://img.shields.io/badge/Architecture-ARM64-orange.svg)](https://github.com/ImL1s/aosp-linux)
[![AVF / crosvm](https://img.shields.io/badge/Hypervisor-AVF_%2F_crosvm_%2F_KVM-purple.svg)](https://source.android.com/docs/core/virtualization)

> **「一個 AOSP 產品，兩個隔離的執行環境，一個統一的使用者體驗」**  
> **"One AOSP Product, Two Isolated Execution Environments, One Unified User Experience."**

> [!NOTE]  
> **Project Status: Advanced Architectural Prototype (進階架構原型)**  
> 本專案為 AOSP 與 Linux 雙執行環境結合的**次世代作業系統進階架構原型**。相關核心組件（`LinuxManagerService`、`linux_bridge`、`LinuxTerminal`）需搭配特定 AOSP 樹進行 Platform Key 簽章與 System App 權限整合。

AOSP Dual-OS 是一款直接修改 AOSP (Android Open Source Project) 打造的次世代行動與桌面融合作業系統。它同時擁有 **完整的 Android 執行環境**（Android Framework, ART, APK, Telephony, AppOps, SystemUI）與 **正統的 GNU/Linux 發行版**（Debian 12 ARM64, 獨立 Kernel 6.6, glibc, systemd, APT, Wayland, Root）。

---

## 🌟 核心特色 (Key Features)

1. **AOSP Native Integration (Host)**
   - 完整的 Android Framework, Binder/AIDL 服務與 Android 15/16 原生相容性。
   - 新增 `LinuxManagerService` (`Context.LINUX_SERVICE`)，支援細粒度 `USE_LINUX_TERMINAL` 與 `MANAGE_LINUX_ENVIRONMENT` 權限控制。

2. **AVF / crosvm / KVM Non-Protected VM (Guest)**
   - 透過 Android Virtualization Framework (AVF) 執行完整 Debian 12 ARM64 Guest。
   - **4-Step HMAC-SHA256 Challenge-Response Vsock 握手**：具備常數時間比對 (`constant_time_eq`) 與 `zeroize` 記憶體擦除。
   - **Android CE Master Key 綁定**：使用 Extract-and-Expand HKDF-SHA256 將 Android PIN 解鎖後的 CE Master Key 衍生為 LUKS2 儲存加密密鑰，鎖屏時全記憶體抹除。

3. **Native Touch Terminal Engine (Touch & IME)**
   - 硬體加速 Surface Canvas 渲染器，JNI 對接 C99 `libvterm` 解析引擎。
   - **CJK IME 多階段組字**：懸浮視窗預覽注音、倉頡與拼音，確定選字後轉為 UTF-8 Byte Stream 派發。
   - **3 種觸控模式**：Shell 模式、TUI Mouse 模式（ANSI SGR 1006 格式 `\e[<0;X;Y;M`，為 Vim/tmux 提供點擊與滾輪）與 Touchpad 模式。

4. **Seamless Wayland GUI Window Forwarding (App Mode)**
   - 將 Guest 側 Wayland (Sommelier/Waypipe) 視窗透過 `virtio-gpu` dma-buf 零拷貝繪製映射至 `LinuxAppProxyActivity`。
   - 每款 Linux GUI 應用（如 VS Code、GIMP、LibreOffice）均映射為獨立的 Android Task，完美整合至 Launcher3 與 Recents 多工切換卡片。

5. **Hardware Portals & Security Hardening**
   - **XDG Hardware Portals Over Vsock**：相機、麥克風、Location 請求受 Android 原生 `LinuxPermissionActivity` 與 AppOps 動態詢問框管制。
   - **Virtiofs SAF 檔案共享**：`LinuxStorageProvider` (SAF DocumentsProvider) 讓 Android 與 Linux `/home` 自由共享檔案。
   - **SELinux Hard NEVERALLOW**：寫死對 `block_device`, `radio`, `efs_file` 以及 `{su init}` 權限轉移的封鎖條款！
   - **Guest A/B EROFS OTA Watchdog**：AVB 2.0 RSA-4096 簽章驗證與開機崩潰自動回滾保護。

---

## 🏗️ 系統架構圖 (Architecture Overview)

```mermaid
graph TD
    subgraph "Android Host Context (AOSP SystemServer)"
        Launcher[Android Launcher3 / Recents]
        TerminalApp[Native Touch Terminal App]
        ProxyAct[LinuxAppProxyActivity]
        
        subgraph "Framework Services"
            LMS[LinuxManagerService]
            LBS[LinuxBridgeService]
            LPS[LinuxPortalService]
            LWS[LinuxWindowBridgeService]
            WMS[WindowManagerService / SurfaceFlinger]
            AppOps[AppOpsManager & PermissionController]
        end
    end

    subgraph "VirtIO Hardware Abstraction"
        Vsock[virtio-vsock (AF_VSOCK Port 5000/5001/5002)]
        VirtFS[virtio-fs (File Share)]
        VirtGPU[virtio-gpu (dma-buf Zero-Copy)]
    end

    subgraph "Linux Guest Context (Debian 12 ARM64 VM)"
        GuestKernel[Linux Kernel 6.6 Mainline]
        Systemd[systemd PID 1]
        
        subgraph "Guest Agents"
            BridgeAgent[android-bridge-agent - Rust]
            PTYAgent[pty-agent]
            WaylandComp[Wayland Proxy / Sommelier]
            PortalAgent[portal-agent]
        end
        
        subgraph "Linux Apps"
            APT[APT Package Manager]
            CLI[Bash / Zsh / Vim / SSH / Git / Rust]
            GUI[VS Code / GIMP / LibreOffice]
        end
    end

    Launcher --> TerminalApp
    Launcher --> ProxyAct
    TerminalApp --> LMS
    ProxyAct --> LWS
    
    LBS <--> Vsock <--> BridgeAgent
    LWS <--> VirtGPU <--> WaylandComp
    LPS <--> AppOps
    
    BridgeAgent --> PTYAgent --> CLI
    WaylandComp --> GUI
    PortalAgent <--> LBS
```

---

## 📂 專案目錄結構 (Repository Structure)

```
aosp-linux/
├── Android.bp                                             # Soong 全域編譯腳本
├── frameworks/base/
│   ├── core/java/android/system/linux/
│   │   ├── ILinuxManager.aidl / ILinuxStatusCallback.aidl # Framework AIDL 介面
│   │   ├── LinuxAppInfo.java                              # Parcelable 資料模型
│   │   └── LinuxManager.java                              # Client API Manager
│   └── services/core/java/com/android/server/linux/
│       └── LinuxManagerService.java                       # SystemServer 核心服務
├── system/
│   ├── linux_bridge/                                      # Host C++ Native Daemon
│   │   ├── linux_bridge_daemon.cpp / .h
│   │   └── vsock_framing.cpp / .h
│   ├── sepolicy/private/
│   │   └── linux_manager.te                               # SELinux 安全條款與 HARD NEVERALLOW
│   └── vold/
│       └── AvbVerifier.cpp                                # AVB 2.0 雙系統簽章校驗
├── packages/apps/LinuxTerminal/                           # Native Touch Terminal 應用
│   ├── Android.bp / AndroidManifest.xml
│   └── src/com/android/virtualization/terminal/
│       ├── TerminalActivity.java                          # 入口 Activity & 快捷列
│       ├── TerminalView.java                              # Canvas 矩陣渲染器
│       └── ime/TerminalInputConnection.java               # CJK IME 組字管道
└── guest/
    └── bridge-agent/                                      # Guest 端 Rust Vsock 守護程序
        ├── Cargo.toml / Cargo.lock
        └── src/main.rs (auth, vsock, ota_rollback)
```

---

## 🛠️ 建置與部署 (Build & Quick Start)

### 1. 複製 repository 至 AOSP 樹中
```bash
git clone https://github.com/ImL1s/aosp-linux.git vendor/aosp-linux
```

### 2. 編譯 AOSP 模組與 Terminal App
```bash
source build/envsetup.sh
lunch aosp_arm64-userdebug

# 編譯 Framework 庫與 Native Daemon
m android.system.linux services.linux linux_bridge_daemon

# 編譯 Touch Terminal App
m LinuxTerminal
```

### 3. 編譯 Guest Rust Agent
```bash
cd guest/bridge-agent
cargo build --release --target aarch64-unknown-linux-gnu
```

### 4. 部署產物 (Build Artifacts)
產出的二進位檔與 APK 將自動歸檔至 `build_out/deployment/`：
- `framework/android.system.linux.jar` & `services.linux.jar`
- `sepolicy/linux_manager.te`
- `apps/LinuxTerminal.apk`
- `guest/android-bridge-agent` & `vbmeta.img` (AVB 2.0)

---

## 🔒 安全模型與條款 (Security & Isolation)

1. **Hypervisor 頁表隔離**：Guest Linux 內的 `root` 僅能管理 Guest VM 內部資源，無法突破 KVM 頁表逃逸。
2. **SELinux HARD NEVERALLOW**：
   - 封鎖對 `block_device` 的 `read/write/open/ioctl` 存取。
   - 封鎖存取 Telephony `radio_data_file` 與 `radio_service`。
   - 嚴禁 Domain Transition 轉移至 `su` 或 `init` 特權進程。
3. **AppOps 動態授權**：Linux App 無法直接存取 Host 硬體，所有 Camera/Mic/Location 請求均必須通過 Host 原生 Permission 對話框詢問使用者。

---

## Support / 支持

If this project saved you some time, you can [buy me a coffee](https://buymeacoffee.com/iml1s).

如果這個專案幫你省了點時間，可以請我喝杯咖啡。

## 📄 授權條款 (License)

本專案採用 [Apache License 2.0](LICENSE) 授權發佈。
