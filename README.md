# THG6500-TAX2 OpenWrt BSP (TGP3)

基于 OpenWrt 的 **THG6500-TAX2**（Triductor TR6560）固件工程。本文档说明 USB/PCIe 问题的根因、已固化修改、编译刷机与日常使用注意事项。

---

## 硬件与 USB 架构

本机 USB **不是** SoC 内置 EHCI/OHCI（`0x10a40000` / `0x10a50000`），而是：

```
SoC PCIe 控制器
  ├─ PCIe0 → WiFi (59e7:0005)
  └─ PCIe1 → ASMedia ASM3042 (3042) → 内核 xHCI → USB 口 / M.2 Key-B
```

- **GPIO 17**：ASM3042（USB3 HUB/控制器）电源/使能  
- **GPIO 27**：M.2 Key-B 槽供电（5G modem 等）  
- **xHCI** 编进内核 `vmlinux`，**没有**独立的 `kmod-usb-xhci` / `kmod-usb3` 模块包  

---

## USB 无法使用的根因（已验证）

| 问题 | 现象 | 说明 |
|------|------|------|
| 走错 USB 主机路径 | `lsusb` 空、`-110`/`-75` | 误启用 SoC EHCI/OHCI 或 `kmod-usb-ehci/ohci/usb2`，与真实硬件（PCIe 3042 + xHCI）不符 |
| 与官方冲突的 USB 内核模块包 | 重复/冲突的 HCD | 启用 `kmod-usb3`、`kmod-usb-core` 等与 **内置 xHCI** 冲突 |
| `tri_pcie.ko` 版本过旧 | 启动时 `PCIE1 Link Down`，`lspci` 无 3042 | 工程内原为 **r1242**，官方固件为 **r1490** |
| `tri_dfx.ko` 与官方不一致 | GPIO/PCIe 时序异常 | 虽同为 r1242 标签，二进制与官方不同 |
| 设备树 `usb_power = <27>` | 与官方 DT 不一致 | 官方 **无** 此属性，GPIO27 仅由 `tri_setup` 控制 |
| 错误 workaround | WiFi 异常、链路反复掉 | `pcie_usb` + `pcie_resume`、`pci rescan`、SoC `tr6560-usb-crg`、`tri-usb-init` 等均无效或有害 |

**结论**：USB 能否用，取决于 **PCIe1 是否在启动时 Link Up** 以及 **内核内置 xHCI**；不是缺一个可 `opkg install` 的 USB 驱动包。

---

## 已固化的修改（请勿随意回退）

以下文件为当前 **正确基线**，重新 `menuconfig` 或从旧分支合并时请注意保留。

### 1. 内核配置

**文件**：`target/linux/tr6560/generic/config-5.10`

- `CONFIG_USB_XHCI_HCD/PCI/PLATFORM=y`（built-in）
- `# CONFIG_USB_EHCI_HCD` / `# CONFIG_USB_OHCI_HCD`（关闭 SoC USB）
- `CONFIG_PCI=y`
- `CONFIG_TUN=y`、`CONFIG_NET_CORE=y`（配合 `etc/init.d/tun`）

**设备树**：`target/linux/tr6560/files-5.10/arch/arm/boot/dts/triductor-tr6560.dtsi`

- `usb_ehci` / `usb_ohci`：`status = "disabled"`

**各机型 DTS**（`THG6500*.dts` 等）：`usb_power = <27>` 已注释，与官方一致，由 `tri_setup` 拉高 GPIO27。

### 2. 厂商内核模块（来自官方固件 5.10.138）

**目录**：`target/linux/tr6560/base-files/lib/modules/5.10.138/`

| 模块 | 版本/说明 |
|------|-----------|
| `tri_pcie.ko` | **r1490**（md5 `d09b12c1…`） |
| `tri_dfx.ko` | 官方同版本二进制（md5 `0d1cad99…`） |

更新方法（官方路由器 IP 示例 `172.18.0.104`）：

```bash
scp root@<官方设备IP>:/lib/modules/5.10.138/tri_pcie.ko \
    target/linux/tr6560/base-files/lib/modules/5.10.138/
scp root@<官方设备IP>:/lib/modules/5.10.138/tri_dfx.ko \
    target/linux/tr6560/base-files/lib/modules/5.10.138/
```

### 3. 启动脚本与模块加载

| 文件 | 作用 |
|------|------|
| `target/linux/tr6560/generic/base-files/etc/init.d/tri_setup` | GPIO **17**（ASM3042）+ **27**（M.2），与官方一致 |
| `target/linux/tr6560/base-files/etc/modules.d/05-tri_bsp` | 含 **`tri_pcie`**，开机自动加载 |
| `target/linux/tr6560/base-files/etc/init.d/tun` | 创建 `/dev/net/tun` |

**已删除、勿恢复**：`etc/init.d/pcie_usb`、`etc/init.d/usb`、`kmod-tr6560-usb-crg`、`tri-usb-init` 等。

### 4. OpenWrt 包配置（`.config`）

须保持关闭（与官方一致，xHCI 在内核内）：

```
# CONFIG_PACKAGE_kmod-usb-core is not set
# CONFIG_PACKAGE_kmod-usb-ehci is not set
# CONFIG_PACKAGE_kmod-usb-ohci is not set
# CONFIG_PACKAGE_kmod-usb2 is not set
# CONFIG_PACKAGE_kmod-usb3 is not set
# CONFIG_PACKAGE_kmod-tr6560-usb-crg is not set
```

可保留用户态工具：`usbutils`、`pciutils`、`libusb-1.0`。

---

## 编译固件

### 环境要求

见 [OpenWrt Build System Setup](https://openwrt.org/docs/guide-developer/build-system/install-buildsystem)。需 Linux、区分大小写文件系统。

### 首次或更新 feeds

```bash
./scripts/feeds update -a
./scripts/feeds install -a
```

### 配置同步（不要用 defconfig 覆盖整份 .config）

```bash
# 同步 Kconfig，保留现有选项；全部回车即可
yes "" | make oldconfig

# 确认 USB 相关包仍关闭
grep -E 'kmod-usb3|kmod-usb-ehci|kmod-usb-core' .config
```

**不要**执行 `make defconfig`，除非你有意重置整份软件包配置。

`make menuconfig` 保存后，请再次确认未勾选 `kmod-usb-ehci/ohci/usb2/usb3`、`kmod-tr6560-usb-crg`。

### 编译

```bash
# 仅改过内核/DTS/模块时
make target/linux/clean

make -j$(nproc)
```

产物通常在 `bin/targets/tr6560/generic/`（具体以 `make menuconfig` 所选机型为准）。

编译时 `harlem` 相关 **WARNING**（缺少 `luci-proto-quectel` 等）可忽略，不影响 THG6500 镜像。

---

## 刷机与验证

### 刷机

按你现有流程烧录 `sysupgrade` 或厂商工具镜像（kernel + rootfs）。改动了 **内核、DTB、内核模块**，需完整刷机，不要只 opkg 升级单个 kmod。

### 启动后检查

```sh
dmesg | grep -iE 'PCIE0|PCIE1|pcie_dev_nr|Link Up'
lspci
lsusb
modinfo tri_pcie | grep version    # 期望 r1490
zcat /proc/config.gz | grep -E 'EHCI|OHCI|XHCI'
```

**正常结果示例**：

- `PCIE0` / `PCIE1 Device Link Up`，`pcie_dev_nr[2]`
- `lspci` 含 `ASMedia Technology Inc. Device 3042`
- `lsusb` 列出 1～2 个 xHCI root hub（无设备时仅 root hub）
- `CONFIG_USB_XHCI_HCD=y`，EHCI/OHCI 为 `not set`

---

## 固件使用说明

### 适用机型

- 本修复主要针对 **THG6500-TAX2**（subtarget `generic`）
- 同系列 DTS 已统一注释 `usb_power`；`norflash` 子机型 `tri_setup` 已同步 GPIO 17/27

### M.2 USB 5G 模组

- 插入前确保固件已刷入本 BSP 修复版
- GPIO 27 在 `tri_setup` 中上电；**无需**再装 `kmod-usb-net-qmi-wwan` 等才能看到 USB 总线（模组驱动另选）
- 若仅做 USB 存储/调试，不插 modem 也应能看到 `lsusb` root hub

### 禁止操作

| 操作 | 后果 |
|------|------|
| `echo 1 > /sys/bus/pci/rescan` | PCIe1 掉链，可能导致 WiFi 心跳异常 |
| 启用 SoC EHCI/OHCI 或 `kmod-usb3` | 与真实硬件路径冲突，USB 再次失效 |
| 从官方单独复制 `xhci-hcd.ko` | 官方 xHCI 在内核内，无此模块，且 vermagic 易不匹配 |
| 恢复 `pcie_usb` / `cs_cli usb_attr_set` | 已验证无效或报错 |

### 故障排查顺序

1. 看 `dmesg` 中 PCIE1 是否 Link Up  
2. `lspci` 是否有 3042  
3. `modinfo tri_pcie` 是否为 r1490  
4. `zcat /proc/config.gz` 中 XHCI/EHCI 配置  
5. 确认未手改 `.config` 打开错误 kmod  

---

## 目录速查

```
target/linux/tr6560/
├── generic/config-5.10          # 内核 USB/PCI/TUN 选项
├── generic/base-files/etc/init.d/tri_setup
├── base-files/etc/modules.d/05-tri_bsp
├── base-files/etc/init.d/tun
├── base-files/lib/modules/5.10.138/tri_pcie.ko   # r1490
├── base-files/lib/modules/5.10.138/tri_dfx.ko
└── files-5.10/arch/arm/boot/dts/
    ├── triductor-tr6560.dtsi      # EHCI/OHCI disabled
    └── THG6500-TAX2.dts           # 等
```

---

## 版本记录

| 日期 | 说明 |
|------|------|
| 2026-05 | USB/PCIe 对齐官方固件：xHCI built-in、tri_pcie r1490、tri_setup GPIO、移除无效 workaround |

---

## 上游 OpenWrt

本仓库基于 OpenWrt 构建系统。通用文档：

- [Quick Start](https://openwrt.org/docs/guide-quick-start/start)
- [Build System](https://openwrt.org/docs/guide-developer/build-system/install-buildsystem)

## License

OpenWrt 部分遵循 GPL-2.0。厂商模块（`tri_*.ko`）版权归 Triductor/原厂商所有。
