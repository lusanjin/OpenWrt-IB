# OpenWrt Auto-Build (Image Builder) - P&W R619AC 128M

![Build Status](https://img.shields.io/badge/build-passing-brightgreen?style=flat-square&logo=githubactions&logoColor=white)
![Target Architecture](https://img.shields.io/badge/target-ipq40xx-blue?style=flat-square&logo=linux&logoColor=white)
![OpenWrt Version](https://img.shields.io/badge/OpenWrt-23.05-00a3e0?style=flat-square&logo=openwrt&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-yellow?style=flat-square)

本项目基于 **GitHub Actions** 和 OpenWrt 官方 **Image Builder** 构建，旨在为 **P&W R619AC 128M（NAND 版）** 提供纯净、高性能的定制固件。

> **为什么选择 Image Builder？**
>
> 区别于传统的源码编译，Image Builder 方案构建速度极快（约 1–2 分钟），且几乎不会遇到复杂的依赖冲突问题，非常适合追求稳定和快速出包的玩家。

## ✨ 固件特性

* **基础版本**：OpenWrt 23.05 稳定版（内核 `5.15`）。
* **纯净精简**：移除冗余组件，基础固件体积仅约 **7 MB**。
* **原生中文**：默认集成简体中文语言包与基础组件，开箱即用。
* **空间拓展**：设定 `ROOTFS_PARTSIZE=64`，刷机后自动分配 64 MB 软件空间，满足后续扩展需求。
* **网络满血**：内置优化脚本，首次开机自动锁定 CPU 为 `performance` 模式，并开启软件流表（Flow Offloading），彻底释放 DHCP 模式下的网络转发性能。

## 🚀 如何使用（构建固件）

1. **Fork 本仓库** 到你的 GitHub 账号下。
2. 进入仓库的 **Actions** 标签页。
3. 在左侧选择 `Build OpenWrt Firmware - P&W R619AC 128M`。
4. 点击右侧的 **Run workflow** 按钮。
5. 等待约 2 分钟，构建完成后在底部的 **Artifacts** 中下载 `r619ac-128m-firmware-tuned` 压缩包。
6. 解压得到 `.bin` 文件，进入路由器后台（**系统 → 备份/升级**），不保留配置刷入即可。

## 🛠️ 插件安装建议

本固件保持纯净，不预装第三方插件。刷机并完成基础网络配置（网络接口、WiFi 设置）后，你可以通过 SSH 连入路由器，自行安装所需的应用（如 PassWall、主题、KMS 服务器等）。

由于不同版本、不同架构的软件源和依赖关系差异较大，**强烈建议不要直接照抄来历不明的命令**。推荐的做法是：在安装前，将以下信息完整地告知 AI 助手（如 ChatGPT、Claude 等），让它为你生成针对性的、经过验证的安装指令：

* 你的路由器型号（`P&W R619AC 128M`）
* 固件版本与架构（`OpenWrt 23.05`，`arm_cortex-a7_neon-vfpv4`）
* 你希望安装的插件名称（例如 `PassWall + Xray`、`Argon 主题`、`vlmcsd` 等）

这样既能避免踩坑，也能确保命令与你的系统完全匹配。

> 💡 **小提示**：在安装任何需要从 GitHub 或海外源下载的插件之前，建议先确保路由器本身已经具备顺畅的海外网络访问能力，否则极大概率会因超时而失败。

## 📝 踩坑与优化记录

### ⚠️ 1. DNS 过滤插件与 PassWall 的冲突

> **注意**：可能导致移动设备无法连接 WiFi

如果你计划在 PassWall 之外再安装 `dnsfilter`、`AdGuard Home`、`MosDNS`、`SmartDNS` 等任何基于 DNS 劫持或过滤的插件，请务必留意一个常见但隐蔽的问题：两者同时开启时，移动设备可能无法获取 IP 地址，表现为“连上 WiFi 但一直卡在正在获取 IP 地址”。

* **冲突机制**：PassWall 自身需要接管系统 DNS（它会启动内部的 DNS 实例并修改 nftables 规则进行重定向），而 DNS 过滤器同样会抢占 53 端口和防火墙重定向规则。两者重复劫持 DNS 流量时，容易产生规则环路，或者导致 `dnsmasq` 进程崩溃。由于 `dnsmasq` 在 OpenWrt 中同时负责 DNS 解析和 DHCP 分配，它一旦罢工，设备就无法从路由器获得 IP 地址——WiFi 信号虽然显示已连接，但网络实际上已经中断。
* **解决方案**：
  * **方案 A（推荐）**：不在外部安装额外的 DNS 过滤插件。PassWall 自带的规则系统已经可以满足去广告需求——在 `PassWall → 规则列表 → 屏蔽列表` 中填入 `geosite:category-ads-all` 即可。
  * **方案 B**：如果确实需要 AdGuard Home 等外部过滤工具，必须在 PassWall 中关闭 DNS 重定向（或将其 DNS 模式设置为不劫持），让 DNS 流量只经过单一入口，避免两者互相干扰。

### ⚠️ 2. 竞斗云2.0网速不达标的两个层面：协议限制与固件调优

#### ① 光猫桥接模式下的 PPPoE 与流表冲突

* **现象**：500M 及以上宽带，使用路由器 PPPoE 拨号时，测速可能只有 200–250 Mbps，远低于预期。
* **根因**：PPPoE 在以太网帧之外增加了一层封装头。当防火墙开启软件流表（Flow Offloading）时，物理 WAN 口会被加入快速转发表。然而，流表规则对 PPPoE 封装报文的处理存在已知缺陷，导致报文无法被正确卸载，被迫回退到 CPU 的普通网络栈进行逐包处理。单核 CPU 的软中断迅速被打满，带宽因此被限制在 200 Mbps 左右。
* **解决方案**：将光猫改为路由模式（由光猫负责 PPPoE 拨号，路由器使用 DHCP 获取 IP）。在 DHCP 模式下，报文没有 PPPoE 封装层，流表可以正常工作，网络性能得以完全释放。

#### ② 固件层面的默认性能限制与编译优化

即使解决了 PPPoE 问题，OpenWrt 对 `ipq40xx` 平台的一些默认设置仍可能限制无线和有线转发性能：

* **CPU 调频策略**：`ipq40xx` 默认使用 `ondemand` 调速器，CPU 在空闲时可能降至 48 MHz，遇到流量突发时升频存在延迟，容易造成抖动和瞬时带宽不足。
* **流表默认未开启**：部分固件版本默认不启用软件流表，所有数据包都走完整内核栈，增加了 CPU 负载。

本项目在编译时已通过注入 `uci-defaults` 脚本解决了这两个问题：**首次开机自动将 CPU 锁定为 `performance` 模式，并开启软件流表（`flow_offloading='1'`，硬件流表保持关闭）**。你无需在刷机后手动 SSH 调整，开箱即是优化后的状态。

> 📌 **特别注意**：上述优化针对的是 **DHCP 模式（光猫拨号）**。如果你将来又把光猫改回桥接模式、由路由器进行 PPPoE 拨号，请务必手动关闭软件流表，否则可能再次触发上述降速问题。

## 💡 其他注意事项

* **为何 PassWall 没有内置在固件中？**
  Image Builder 难以直接处理第三方预编译包的复杂依赖关系。先刷入纯净固件，再通过 AI 辅助生成针对性的 SSH 命令进行安装，容错率最高，也最不容易导致系统变砖。

* **dnsmasq 版本警告说明**
  OpenWrt 23.05 官方源稳定版为 `2.90`。PassWall 可能会提示建议升级至 `2.92` 以增强稳定性。在 23.05 环境下，2.90 已修复关键安全漏洞，日常代理与解析完全够用，可安全忽略此提示。

> [!WARNING]
> **免责声明**：刷机有风险，操作需谨慎。本项目仅供学习与交流使用。
