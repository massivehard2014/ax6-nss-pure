# Redmi AX6 · LibWrt NSS 纯净版云端编译

给红米 AX6（IPQ8071A / IPQ807X）编译一套**满血 NSS 硬件加速 + 无第三方插件**的 OpenWrt 系统。
源码使用你指定的 `https://github.com/LiBwrt/LibWrt`，分支 `25.12-nss`（该仓库当前唯一分支，专为 IPQ807X/IPQ60XX 做满血 NSS）。

## 为什么用这套配置，而不是直接 fork 现成云编译模板

LibWrt 官方 README 推荐了 4 个云编译项目（breeze303/openwrt-ci、ZqinKing/wrt_release、
laipeng668/openwrt-ci-roc、VIKINGYFY/OpenWRT-CI）。但实测有两个不符合你的需求：

1. **源码不一致**：这些模板大多 clone 的是老仓库 `LiBwrt/openwrt-6.x` 的 `main` 分支，
   不是你指定的 `LiBwrt/LibWrt` 的 `25.12-nss` 分支。
2. **并不纯净**：它们的 config 默认塞满了 passwall / sing-box / xray / ddns-go 等插件。

本仓库的 `.github/workflows/build.yml` 直接 clone 你指定的仓库和分支，配 `configs/redmi-ax6-pure.config`，
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

进入仓库 **Actions** 标签 → 左侧选 `Build LibWrt NSS Pure - Redmi AX6` → **Run workflow**。
首次约 1.5～3 小时（GitHub  runners 性能波动较大）。

### 3. 取固件

两种方式都可用：
- 仓库 **Releases**：Tag `AX6-NSS-PURE`（含 sha256sums，推荐）
- 该次 Actions 运行页底部的 **Artifacts**：`AX6-NSS-PURE`

产物文件名形如 `openwrt-qualcommax-ipq807x-redmi_ax6-stock-squashfs-sysupgrade.bin`。

> 注：本配置针对 **custom U-Boot layout**（已刷第三方 U-Boot）编译 `redmi_ax6-stock` 变体。
> 该变体只产出 sysupgrade 固件，**不再生成 `initramfs-factory.ubi`**，因为你的设备已有可用 U-Boot，
> 不需要走"原厂 ubiformat 过渡"那条路。

### 4. 校验

下载后核对 `sha256sums`，例如 Windows PowerShell：

```powershell
certutil -hashfile .\固件文件名.bin SHA256
```

## 刷机提醒（重要）

你的 AX6 是 **custom U-Boot layout**，所以刷法与原厂布局不同：

- **不需要也不能**走原厂那套"降级 → 开 SSH → `ubiformat` 写 initramfs → `nvram set flag_boot_rootfs`"流程。
  那套是给纯原厂固件设备用的，你的分区表已经被第三方 U-Boot 改过，套用会出错。
- 正确做法：进入你现有的 U-Boot 恢复界面（或当前 OpenWrt 的 `sysupgrade`），
  直接刷 `openwrt-qualcommax-ipq807x-redmi_ax6-stock-squashfs-sysupgrade.bin`。
- 刷机前确认你的 U-Boot 支持的分区布局与本固件匹配（第三方 U-Boot 有多种版本，分区大小可能不同）。
  **不确定就先备份当前固件和 U-Boot 环境变量，刷机有变砖风险。**
- 已知小坑：sysupgrade 偶尔因 wpad 未停导致失败，升级前先在 SSH 里执行 `service wpad stop`。

刷机涉及硬件风险，本文只给编译方案；具体刷机命令请以你的 U-Boot 文档为准。

## 关于 NSS

- 本配置保留了 IPQ807X 的 NSS 驱动（`kmod-qca-nss-*`）与固件（`nss-firmware-ipq8074`），
  2.4G / 5G WiFi offload、PPPoE、桥接、VLAN 等均走 NSS 加速。
- **不要再在防火墙里勾选硬件/软件卸载加速**，会和 NSS 冲突。
- 验证：跑大流量时 CPU 占用很低，或 `lsmod | grep qca_nss` 看驱动已加载。

## 想加插件怎么办

编辑 `configs/redmi-ax6-pure.config`，按 OpenWrt 的 `CONFIG_PACKAGE_xxx=y` 规则加上你要的包，
再重新 Run workflow 即可。例如要装 passwall，把文件末尾对应的 `# CONFIG_PACKAGE_luci-app-passwall is not set`
改成 `CONFIG_PACKAGE_luci-app-passwall=y`。但那就不是纯净版了。

## 想改默认 IP / 加自定义文件

在仓库根目录建 `files/etc/config/network` 覆盖默认配置即可；workflow 已预留把 `files/` 拷进源码的逻辑
（如需请自行在 "载入纯净版配置" 步骤加一行 `[ -d files ] && cp -a files $OPENWRT_PATH/`）。
