OpenWrt Auto-Build (Image Builder) - P&W R619AC 128M
本项目基于 GitHub Actions 和 OpenWrt 官方 Image Builder 构建针对 P&W R619AC 128M（NAND 版）的纯净、高性能定制固件。

区别于传统的源码编译，Image Builder 方案构建速度极快（约 1-2 分钟），且几乎不会遇到复杂的依赖冲突问题，非常适合追求稳定和快速出包的玩家。

✨ 固件特性
基础版本：OpenWrt 23.05 稳定版（内核 5.15）。

纯净精简：移除冗余组件，基础固件体积仅约 7MB。

原生中文：默认集成简体中文语言包与基础组件，开箱即用。

空间拓展：设定 ROOTFS_PARTSIZE=64，刷机后自动分配 64MB 软件空间，满足后续扩展。

网络满血：内置优化脚本，首次开机自动锁定 CPU 为 performance 模式，并开启软件流表（Flow Offloading），彻底释放 DHCP 模式下的网络转发性能。

🚀 如何使用（构建固件）
Fork 本仓库 到你的 GitHub 账号下。

进入仓库的 Actions 标签页。

在左侧选择 Build OpenWrt Firmware - P&W R619AC 128M。

点击右侧的 Run workflow 按钮。

等待约 2 分钟，构建完成后在底部的 Artifacts 中下载 r619ac-128m-firmware-tuned 压缩包。

解压得到 .bin 文件，进入路由器后台（系统 -> 备份/升级），不保留配置刷入即可。

🛠️ 高级配置与插件安装 (SSH)
刷机并完成基础网络配置（宽带拨号、WiFi 设置）后，可通过 SSH 连入路由器，执行以下命令安装必备插件。

⚠️ 注意：以下命令请在路由器已连通海外网络，或能顺畅访问 GitHub 的前提下执行，以避免网络超时。

1. 安装 PassWall 及核心引擎
bash
# 替换 DNS 组件
opkg update
opkg remove dnsmasq --force-depends
opkg install dnsmasq-full

# 导入 PassWall GPG 公钥并添加源
wget -O /tmp/passwall.pub https://master.dl.sourceforge.net/project/openwrt-passwall-build/ipk.pub
opkg-key add /tmp/passwall.pub
read release arch << EOF
$(. /etc/openwrt_release ; echo ${DISTRIB_RELEASE%.*} $DISTRIB_ARCH)
EOF
for feed in passwall_luci passwall_packages passwall2; do
  echo "src/gz $feed https://master.dl.sourceforge.net/project/openwrt-passwall-build/releases/packages-$release/$arch/$feed" >> /etc/opkg/customfeeds.conf
done

# 安装 PassWall、Xray 及透明代理依赖
opkg update
opkg install luci-app-passwall luci-i18n-passwall-zh-cn xray-core geoview luci-compat kmod-nft-socket kmod-nft-tproxy
2. 安装 Argon 主题
bash
wget --no-check-certificate -O /tmp/luci-theme-argon.ipk https://ghfast.top/https://github.com/jerrykuku/luci-theme-argon/releases/download/v2.3.2/luci-theme-argon_2.3.2-r20250207_all.ipk
opkg install /tmp/luci-theme-argon.ipk
3. 安装 KMS 服务器 (vlmcsd)
bash
# 下载核心与界面
wget -O /tmp/vlmcsd.ipk https://github.com/cokebar/openwrt-vlmcsd/raw/gh-pages/vlmcsd_svn1113-1_arm_cortex-a7_neon-vfpv4.ipk
wget -O /tmp/luci-app-vlmcsd.ipk https://github.com/cokebar/luci-app-vlmcsd/releases/download/v1.0.2-1/luci-app-vlmcsd_1.0.2-1_all.ipk

# 安装核心
opkg install --force-depends --force-checksum /tmp/vlmcsd.ipk

# 暴力解压安装界面（绕过架构限制）
mkdir -p /tmp/vlmcsd-extract && cd /tmp/vlmcsd-extract
tar -xzf /tmp/luci-app-vlmcsd.ipk && tar -xzf data.tar.gz -C /
cd /tmp && rm -rf /tmp/vlmcsd-extract

# 启动并放行防火墙
/etc/init.d/vlmcsd enable && /etc/init.d/vlmcsd start
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-KMS'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].dest_port='1688'
uci set firewall.@rule[-1].target='ACCEPT'
uci commit firewall && /etc/init.d/firewall restart
4. 刷新后台与启用广告过滤
bash
# 刷新后台缓存
rm -rf /tmp/luci-*
/etc/init.d/rpcd restart
/etc/init.d/uhttpd restart
界面操作：进入 LuCI 后台，在 系统 -> 语言与样式 切换为 Argon 主题。

去广告：进入 服务 -> PassWall -> 规则列表 -> 屏蔽列表，填入 geosite:category-ads-all，点击保存并应用。

📝 踩坑与优化记录
⚠️ 关于光猫桥接模式导致的网速断崖式下跌
许多用户在 500M 及以上带宽的环境下，会遇到路由器 PPPoE 拨号后网速腰斩（如只有 200Mbps）的情况。经过深度排查，这与 ipq40xx 平台的底层机制密切相关：

现象：光猫桥接 -> 路由器 PPPoE 拨号，测速最高仅 200Mbps - 250Mbps。无论开启或关闭软件流表（Flow Offloading），速度均无法突破。

根因：

PPPoE 协议在以太网帧外增加了一层封装头。
当开启软件流表（flow_offloading='1'）时，防火墙将物理 WAN 口加入快速转发表（flowtable）。然而，流表无法正确处理带有特殊封装的 PPPoE 报文，导致解包异常。
报文被迫回退到 CPU 的普通网络栈处理，形成单核性能瓶颈，导致带宽受限。
解决方案：将光猫改为路由模式（DHCP 模式），由光猫负责拨号，路由器负责 NAT 转发与 WiFi。在此模式下，报文没有额外封装，配合本项目内置的开启软件流表脚本，网速可直接跑满 500M 甚至 600M+ 宽带。

💡 其他注意事项
为何 PassWall 没有内置在固件中？
Image Builder 难以直接处理第三方预编译包的复杂依赖关系。先刷入纯净固件，再通过 SSH 联网安装，容错率最高，也最不容易导致系统变砖。

dnsmasq 版本警告说明：
OpenWrt 23.05 官方源稳定版为 2.90。PassWall 可能会提示建议升级至 2.92 以增强稳定性。在 23.05 环境下，2.90 已修复关键安全漏洞，日常代理与解析完全够用，可安全忽略此提示。

Disclaimer: 刷机有风险，操作需谨慎。本项目仅供学习与交流使用。
