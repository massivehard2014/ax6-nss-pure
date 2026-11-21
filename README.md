# Redmi AX6 · LibWrt NSS 纯净版云端编译

给红米 AX6（IPQ8071A / IPQ807X）编译一套**满血 NSS 硬件加速 + 无第三方插件**的 OpenWrt 系统。

源码：`https://github.com/LiBwrt/LibWrt`，分支 `25.12-nss`（该仓库当前唯一分支，专为 IPQ807X / IPQ60XX 做满血 NSS）。

## 设计取舍

LibWrt 官方 README 推荐了 4 个云编译项目（breeze303/openwrt-ci、ZqinKing/wrt_release、
laipeng668/openwrt-ci-roc、VIKINGYFY/OpenWRT-CI），但存在两个问题：

1. **源码不一致**：这些模板大多 clone 老仓库 `LiBwrt/openwrt-6.x` 的 `main` 分支，
   不是 `LiBwrt/LibWrt` 的 `25.12-nss` 分支。
2. **并不纯净**：默认配置塞满了 passwall / sing-box / xray / ddns-go 等插件。

本仓库的 `.github/workflows/build.yml` 直接 clone 上述仓库与分支，配 `configs/redmi-ax6-pure.config`，
只保留：系统基础 + WiFi(ath11k) + NSS 全套驱动/固件 + LuCI 中文。**不含任何代理类插件**。

## 使用步骤

### 1. 建仓库

在 GitHub 新建一个**公开**仓库（名字随意，例如 `ax6-nss-pure`），把本目录的文件原样传上去，保持结构：

```
.github/workflows/build.yml
configs/redmi-ax6-pure.config
README.md
```

> Actions 需要可写权限发布 Release：仓库 Settings → Actions → General → Workflow permissions 勾选 "Read and write permissions"。

### 2. 触发编译

两种方式：

- **手动**：进入仓库 **Actions** 标签 → 左侧选 `Build LibWrt NSS Pure - Redmi AX6` → **Run workflow**。
- **定时**：每月 1 号北京时间 08:00 自动编译一次（workflow 中的 `schedule` 触发器）。

首次编译约 1.5～3 小时，取决于 GitHub Actions runner 的性能。

### 3. 取固件

两种方式都可用：

- 仓库 **Releases**：Tag `AX6-NSS-PURE`（含 sha256sums，推荐）
- 对应 Actions 运行页底部的 **Artifacts**：`AX6-NSS-PURE`

Release 中包含以下产物（以 `redmi_ax6-stock` 变体为例）：

| 文件 | 用途 |
|---|---|
| `...-redmi_ax6-stock-squashfs-sysupgrade.bin` | 已在 OpenWrt / 第三方 U-Boot 环境下直接 sysupgrade |
| `...-redmi_ax6-stock-squashfs-factory.ubi` | 第三方 U-Boot 界面刷入的 factory 固件 |
| `...-redmi_ax6-stock-initramfs-uImage.itb` | 临时系统镜像 |

> 本配置针对 **custom U-Boot layout**（已刷第三方 U-Boot、分区合并为大分区）编译 `redmi_ax6-stock`
> 变体。该变体产出 `sysupgrade.bin` 和 `factory.ubi`，不产出 `initramfs-factory.ubi`
> （面向已有可用 U-Boot 的设备，无需走原厂 ubiformat 过渡流程）。

### 4. 校验

下载后核对 `sha256sums`，例如 Windows PowerShell：

```powershell
certutil -hashfile .\固件文件名.bin SHA256
```

## 刷机须知（重要）

- 本固件是 `redmi_ax6-stock` 变体，适配 **custom U-Boot layout**（第三方 U-Boot + 合并大分区）。
- **完全原厂未刷机的 AX6 不能直接刷本固件**：原厂是双 35M 小分区布局，必须先开 SSH、刷过渡固件、
  更换大分区 U-Boot 完成分区扩容。跨布局刷写会导致变砖。
- 更换 U-Boot / 扩容分区前**务必先备份原厂分区**，尤其是 `ART`（无线校准数据，每台设备唯一，
  丢失后 WiFi 无法恢复）和 U-Boot 本身。
- 刷机有变砖风险，请自行评估并准备好救砖手段。
- **详细刷机流程参考**：https://www.right.com.cn/forum/thread-8426229-1-1.html

## 关于 NSS

- 本配置保留了 IPQ807X 的 NSS 驱动（`kmod-qca-nss-*`）与固件（`nss-firmware-ipq807x`），
  2.4G / 5G WiFi offload、PPPoE、桥接、VLAN 等均走 NSS 加速。
- 不要在防火墙里勾选硬件/软件卸载加速，会和 NSS 冲突。
- 验证方法：跑大流量时 CPU 占用很低，或 `lsmod | grep qca_nss` 确认驱动已加载。

## 想加插件怎么办

编辑 `configs/redmi-ax6-pure.config`，按 OpenWrt 的 `CONFIG_PACKAGE_xxx=y` 规则加上需要的包，
再重新 Run workflow 即可。例如把文件末尾的 `# CONFIG_PACKAGE_luci-app-passwall is not set`
改成 `CONFIG_PACKAGE_luci-app-passwall=y`。加入插件后不再是纯净版。

## 想改默认 IP / 加自定义文件

在仓库根目录建 `files/etc/config/network` 覆盖默认配置即可；workflow 已预留把 `files/` 拷进源码的逻辑。
