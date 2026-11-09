# OpenCore EFI · Alder Lake-H MoDT · Intel Iris Xe (8086:46A0)

OpenCore 1.0.8 EFI for a white-label **MoDT (mobile-on-desktop)** board carrying an Alder Lake-H engineering-sample CPU, running **macOS Tahoe 26.7.2** with the Intel Iris Xe iGPU driven by [WhateverXe](https://github.com/b00t0x/WhateverXe). The NVIDIA CMP 30HX on this board is hidden from macOS.

> **This is not a universal EFI.** It is tuned for one specific board and one specific CPU sample. Treat it as a reference, not a drop-in solution.

---

## ⚠️ 使用前必读

1. **必须自己生成 SMBIOS。** 仓库里的 `SystemSerialNumber`、`MLB`、`SystemUUID`、`ROM` 全是占位符。用 [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) 生成一套属于你的，填进 `PlatformInfo → Generic`，否则 iMessage / App Store 会出问题，多人共用同一套标识也会互相打架。
2. **本仓库不含任何 Apple 专有二进制。** HFS+ 驱动使用 OpenCore 自带的开源 `OpenHfsPlus.efi`；macOS 安装镜像请自行从 Apple 获取。
3. **不要期待开箱即用。** 特别是显示输出，见下方「已知限制」。

---

## 硬件

| 项目 | 型号 / 参数 |
| --- | --- |
| 主板 | 白牌 MoDT 板（mobile-on-desktop），iGPU 直连显示输出 |
| CPU | Intel Core i7-12700H **工程样品**（系统内显示 `Genuine Intel(R) 0000`，6P+8E / 20 线程） |
| 核显 | Intel Iris Xe，`8086:46A0`，revision `0x00`，subsystem `8086:2212`，96 EU |
| 独显 | NVIDIA CMP 30HX `10de:2189`（已通过 `disable-gpu` 从 macOS 屏蔽） |
| 内存 | 16 GB (2×8 GB DDR4-2666) |
| 存储 | Micron MTFDKBA1T0TFH 1 TB NVMe（Windows + macOS 双系统） |
| 有线网卡 | Realtek RTL8168 (`10ec:8168`) |
| 无线 | Intel Wi-Fi 6E AX211 (`8086:51f0`) |
| 显示器 | TSQ2708，2560×1440 @ 60 Hz（HDMI / DP 双输入） |

## 软件版本

| 项目 | 版本 |
| --- | --- |
| macOS | Tahoe 26.7.2 (25G332) |
| OpenCore | 1.0.8 RELEASE |
| WhateverXe | 0.8.0 RELEASE |
| WhateverXe Graphics | 0.4.1（`AppleIntelTGLCompat 0.8.0`，`AppleIntelTGLGraphics/Framebuffer 16.0.0`） |
| Lilu | 1.7.3 RELEASE |
| AMFIPass | 1.4.1 Mod（WEX 专用，非上游版本） |
| `csr-active-config` | `03080000` (0x803) |

---

## 目录结构

```
EFI/
├── BOOT/
│   └── BOOTx64.efi
└── OC/
    ├── ACPI/            # SSDT-EC / PLUG-ALT / RTCAWAC / SBUS / MCHC / USBX / USB-Reset
    ├── Drivers/         # OpenHfsPlus.efi, OpenRuntime.efi, ResetNvramEntry.efi
    ├── Kexts/           # 见下方第三方组件清单
    ├── Resources/       # OpenCore 启动界面资源
    ├── config.plist          ← 默认配置：核显加速模式
    ├── config-install.plist  ← 备用配置：安装 / 排错模式
    └── OpenCore.efi
```

### 两份配置的区别

| 配置 | WhateverXe | 用途 |
| --- | --- | --- |
| `config.plist` | 启用 | 日常使用，核显加速生效。**要求已安装 WhateverXe Graphics 包，且 macOS ≥ 26.7** |
| `config-install.plist` | 关闭 | 安装 macOS、首次启动或加速出问题时回退。用基础帧缓冲，能进桌面但没有 Metal |

切换方式：把想用的那份改名成 `config.plist`，另一份留作备份即可。

---

## 使用方法

1. 准备一个 FAT32 分区，把整个 `EFI` 文件夹放在根目录。
2. BIOS 中关闭 CSM；Secure Boot 视主板而定（本机为关闭）。VT-d 可以保持开启，EFI 里已设 `DisableIoMapper = true`。
3. 核显 DVMT Pre-Allocated 设为 96 MB（本机为 96 MB）。
4. 用 GenSMBIOS 生成 SMBIOS，替换 `config.plist` 与 `config-install.plist` 里的四个字段。
5. 从 UEFI 启动菜单选择该设备，进 OpenCore 后选 macOS。

安装完系统、升级到 **26.7 或更高** 之后，再安装 [WhateverXe Graphics](https://github.com/b00t0x/WhateverXe-Graphics/releases) 包并重启，加速才会生效。顺序反了不会有效果。

---

## 已知限制

这些是实测出来的边界，不是配置错误：

- **DisplayPort 输出无法启用加速。** 本板 DP 走 DDI B，WhateverXe 当前的准入逻辑只承认 HDMI，`bootDDI()` 返回 `0xFFFFFFFF`，系统退回 `IONDRVFramebuffer`。详情见 [WhateverXe#73](https://github.com/b00t0x/WhateverXe/issues/73)。
- **HDMI 下需要修正 EDID，否则画面发紫、出现色带。** 显示器若声明 YCbCr / 深色，驱动会照做。详见 [WhateverXe#74](https://github.com/b00t0x/WhateverXe/issues/74)。
- **系统休眠 / 唤醒仍是实验性功能**，建议先在「节能」里关闭自动休眠。
- **工程样品 CPU**：CPUID 被伪装成 Comet Lake 才能让 macOS 接受，不同批次样品行为可能不同。
- **30HX 只是从 macOS 屏蔽**，不代表物理断电，功耗与发热依旧。
- 只在 macOS 26.7.2 上验证过；同系列其他版本可能需要相应调整 kext 或 boot-args。

---

## 第三方组件

本仓库分发的第三方组件均保留其原始许可，来源如下表。使用或再分发前请自行核对上游许可证。

| Kext | 版本 | Bundle ID | 上游 |
| --- | --- | --- | --- |
| AirportItlwm.kext | 2.4.0 | com.zxystd.AirportItlwm | OpenIntelWireless/itlwm |
| AMFIPass.kext | 1.4.1 | com.dhinakg.AMFIPass | dhinakg/AMFIPass（1.4.1 Mod） |
| BlueToolFixup.kext | 2.7.3 | as.acidanthera.BlueToolFixup | acidanthera/BrcmPatchRAM |
| BrightnessKeys.kext | — | as.acidanthera.BrightnessKeys | acidanthera/BrightnessKeys |
| CpuTopologyRebuild.kext | 2.0.2 | org.b00t0x.CpuTopologyRebuild | 见 Info.plist |
| IntelBluetoothFirmware.kext | 2.5.1 | com.zxystd.IntelBluetoothFirmware | OpenIntelWireless/IntelBluetoothFirmware |
| IntelBTPatcher.kext | 2.5.1 | com.zxystd.IntelBTPatcher | OpenIntelWireless/IntelBluetoothFirmware |
| itlwm.kext | 2.3.0 | com.zxystd.itlwm | OpenIntelWireless/itlwm |
| Lilu.kext | 1.7.3 | as.vit9696.Lilu | acidanthera/Lilu |
| NVMeFix.kext | 1.1.4 | org.acidanthera.NVMeFix | acidanthera/NVMeFix |
| RealtekRTL8111.kext | 2.4.2 | com.insanelymac.RealtekRTL8111 | Mieze/RealtekRTL8111 |
| RestrictEvents.kext | 1.1.7 | as.vit9696.RestrictEvents | acidanthera/RestrictEvents |
| SMCBatteryManager.kext | 1.3.9 | ru.usrsse2.SMCBatteryManager | acidanthera/VirtualSMC |
| SMCDellSensors.kext | 1.3.9 | as.lvs1974.SMCDellSensors | acidanthera/VirtualSMC |
| SMCLightSensor.kext | 1.3.9 | ru.usrsse2.SMCLightSensor | acidanthera/VirtualSMC |
| SMCProcessor.kext | 1.3.9 | as.vit9696.SMCProcessor | acidanthera/VirtualSMC |
| SMCSuperIO.kext | 1.3.9 | ru.joedm.SMCSuperIO | acidanthera/VirtualSMC |
| USBToolBox.kext | — | com.dhinakg.USBToolBox.kext | USBToolBox/USBToolBox |
| UTBDefault.kext | 1.0 | com.dhinakg.USBToolBox.injector | 由 USBToolBox 生成 |
| VirtualSMC.kext | 1.3.9 | as.vit9696.VirtualSMC | acidanthera/VirtualSMC |
| VoodooI2C.kext | 2.9.1 | com.alexandred.VoodooI2C | VoodooI2C/VoodooI2C |
| VoodooI2CHID.kext | 1.0 | com.alexandred.VoodooI2CHID | VoodooI2C/VoodooI2C |
| VoodooPS2Controller.kext | 2.3.8 | as.acidanthera.voodoo.driver.PS2Controller | acidanthera/VoodooPS2 |
| WhateverXe.kext | 0.8.0 | org.b00t0x.WhateverXe | b00t0x/WhateverXe |
| XHCI-unsupported.kext | 0.9.2 | org.rehabman.injector.XHCI.unsupported | 见 Info.plist |

- **OpenCore** 本体与 `OpenHfsPlus.efi`、`OpenRuntime.efi`、`ResetNvramEntry.efi` 来自 [acidanthera/OpenCorePkg](https://github.com/acidanthera/OpenCorePkg)，BSD-3-Clause，全文见 `LICENSE`。
- **WhateverXe** 继承 WhateverGreen 的 BSD-3-Clause 版权声明，再分发时请保留。
- `Resources/` 下的启动界面资源来自 [acidanthera/OcBinaryData](https://github.com/acidanthera/OcBinaryData)。

---

## 免责声明

本项目仅用于个人硬件研究与学习。修改固件设置、分区和引导项存在数据丢失风险，请自行备份。作者不对任何直接或间接损失负责，也不提供任何形式的担保。
