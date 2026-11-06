# 红米 AX6000（RB06）ImmortalWrt 25.12.2 自编译

设备：Xiaomi Redmi Router AX6000 · MT7986A · 512MB · 128MB SPI-NAND
引导：hanwckf 不死 U-Boot（multi-layout，WebUI `192.168.31.1`）

## ⚠️ 布局必须与镜像配对（唯一会变砖的地方）

| U-Boot 里的 `mtd_layout_label` | 分区 | 用哪个镜像 |
|---|---|---|
| `default`（官方原厂布局） | `ubi_kernel 30720k` + `ubi 81920k` | **本仓库产物**，或官方 `...-ax6000-stock-*` |
| `immortalwrt-110m`（本机旧布局） | 单 `ubi 110m` | 本仓库不产出；需要时另行编译 |
| （OpenWrt U-Boot 布局） | `ubi@0x580000 0x7a80000` | 官方 `...-ax6000-ubootmod-*.itb`（本仓库不涉及） |

本仓库只出 **stock（官方原版）布局 + 官方主线 mt76 开源驱动** 的镜像，不含任何闭源驱动产物。

## 工作流

### 1. `build-ax6000-stock.yml`（唯一工作流，push 或手动触发）

- 基座：`immortalwrt/immortalwrt` **v25.12.2**（内核 6.12，mt76 开源驱动，apk 包管理）
- 包源：`packages` / `luci` / `routing` / `telephony` / `video` 全部跟踪 `openwrt-25.12` **分支最新提交**（取 25.12 线上最新版本）
- OpenClash：`vernesong/OpenClash` **master 最新**（当前 0.48.002）
- 设备：`xiaomi_redmi-router-ax6000-stock`（仅此一台，避免误编 ubootmod）

三件套（`configs/ax6000-stock.config` 中显式勾选）：

| 需求 | 包 | 说明 |
|---|---|---|
| OpenClash | `luci-app-openclash` + `ruby` `ruby-yaml` `kmod-tun` `kmod-nft-tproxy` `kmod-inet-diag` | ruby 依赖约 15~20MB，注意 overlay 余量 |
| WireGuard | `kmod-wireguard` `wireguard-tools` `luci-proto-wireguard` `qrencode` | 内核模块随内核同源编译，无 ABI 问题 |
| UPnP | `luci-app-upnp` + `miniupnpd-nftables`（当前 2.3.9） | 25.12 默认 firewall4，选 nftables 变体 |
| 主题 | `luci-theme-argon` 2.4.7 + `luci-app-argon-config` | 开机即为默认主题（`files/etc/uci-defaults/99-default-theme-argon` 注入） |

产物（Actions → Artifacts）：

```
immortalwrt-25.12.2-mediatek-filogic-xiaomi_redmi-router-ax6000-stock-squashfs-sysupgrade.bin   ← 刷这个
immortalwrt-25.12.2-mediatek-filogic-xiaomi_redmi-router-ax6000-stock-initramfs-factory.ubi     ← 兜底
immortalwrt-25.12.2-mediatek-filogic.manifest                                                   ← 包清单
SHA256SUMS.txt · VERSIONS.txt                                                                   ← 校验 + 版本存证
```

## 刷入（摘要）

1. 电脑网线接 LAN 口，静态 IP `192.168.31.100/24`
2. 断电 → 按住 RESET ≥15 秒上电 → `http://192.168.31.1`
3. 布局选 **`default`** + 上传 `...-stock-squashfs-sysupgrade.bin` → 开始刷机
4. 刷完 PC 改回 DHCP，访问 `http://192.168.1.1`（root / 无密码）

回退：同一入口，布局保持 `default` + 刷官方 `24.10.6` 的 stock 镜像（opkg 体系，可在线下载）。

## 注意

- 本仓库不含任何密钥/密码，仅构建配置。
- Argon 主题已编译进镜像并在首次启动设为默认（主题包自带的 `30_luci-theme-argon` + 本仓库的 `99-default-theme-argon`）。
- `Factory`/`Nvram`/`Bdata`/`BL2`/`FIP` 全程不写，不会黑砖。
- 若 ISP 绑定了 WAN MAC，升级后需手工改回旧 MAC（Factory 里读到的是 `24:cf:24:5d:6d:3f/.40`）。
