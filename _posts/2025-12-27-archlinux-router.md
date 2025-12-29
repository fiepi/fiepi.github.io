---
layout: post
title: 终极指南：从零打造基于 N100 + Arch Linux 的 2.5G 全能软路由
description: 终极指南：从零打造基于 N100 + Arch Linux 的 2.5G 全能软路由
date: 2025-12-27
category: archlinux
keywords: archlinux, v2ray, nftables, router
---

本文将详细记录如何使用 **Intel N100** 迷你主机（配备 4 个 i226-V 2.5G 网口）和 **Arch Linux** 搭建一台高性能软路由。

**核心特性：**

1. **性能榨干**：开启 `Flow Offloading` 硬件加速，跑满 2.5G 带宽 CPU 占用极低。
2. **Android 兼容**：通过精细的 IPv6 RA 调优，解决 Android 锁屏断流问题。
3. **架构解耦**：基础防火墙与透明代理分离，代理挂了不影响全屋正常上网。
4. **无感代理**：基于 TProxy 技术，支持 UDP 转发（游戏优化）。

---

## 准备工作

* **硬件**：Intel N100 主机，4 个网口。
* **系统**：已安装好的 Arch Linux（建议安装 `base`, `linux`, `linux-firmware`, `vim`, `intel-ucode`, `ethtool`, `nftables`, `v2ray`, `ppp` 等基础包）。
* **权限**：全程使用 root 用户操作。

---

## 第一步：系统引导与内核优化

### 1.1 优化 Systemd-boot 引导参数

Intel i226-V 网卡在 Linux 下可能因 ASPM 省电机制导致连接不稳定。我们需要禁用它。

编辑 Systemd-boot 的条目文件（通常在 `/boot/loader/entries/` 下，例如 `arch.conf`）：

```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options root=LABEL=arch_os rootflags=subvol=@ rw rootfstype=btrfs
# 追加 pcie_aspm=off 参数
options pcie_aspm=off

```

### 1.2 内核参数调优 (Sysctl)

创建文件 `/etc/sysctl.d/99-router.conf`，写入以下内容以开启转发并优化高并发性能：

```ini
# --- 开启转发 ---
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1

# --- 拥塞控制 (BBR) ---
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr

# --- 2.5G 网卡缓冲区优化 ---
net.core.rmem_max=16777216
net.core.wmem_max=16777216
net.ipv4.tcp_rmem=4096 87380 16777216
net.ipv4.tcp_wmem=4096 87380 16777216
net.ipv4.tcp_fastopen=3

# --- 连接跟踪 (解决 UDP 断流/游戏掉线) ---
# 默认 UDP 超时只有 30s，改为 180s 防止语音通话中断
net.netfilter.nf_conntrack_udp_timeout = 180
net.netfilter.nf_conntrack_udp_timeout_stream = 180
# 确保 TCP 连接稳定
net.netfilter.nf_conntrack_tcp_timeout_established = 432000

```

---

## 第二步：构建基础网络 (Systemd-networkd)

我们使用 MAC 地址绑定网卡名称，确保重启后接口不会乱序。

### 2.1 固定网卡名称

查看你的网卡 MAC 地址 (`ip link`)，然后创建对应的 `.link` 文件。
目录：`/etc/systemd/network/`

**`10-wan0.link` (拨号口)**

```ini
[Match]
MACAddress=xx:xx:xx:xx:xx:01
[Link]
Name=wan0

```

**`11-lan0.link` (局域网口1)**

```ini
[Match]
MACAddress=xx:xx:xx:xx:xx:02
[Link]
Name=lan0

```

*(如果有 lan1, lan2，请依样画葫芦创建)*

### 2.2 创建虚拟网桥

**`20-br-lan.netdev`**

```ini
[NetDev]
Name=br-lan
Kind=bridge
# 固定 MAC 地址，防止 IPv6 Link-Local 地址变动
MacAddress=xx:xx:xx:xx:xx:05

[Bridge]
# 关闭 Snooping，防止误杀 IPv6 RA 广播
MulticastSnooping=no
MulticastQuerier=yes

```

### 2.3 绑定物理网口

**`30-wan0.network`** (WAN 口仅启动，不配 IP)

```ini
[Match]
Name=wan0
[Link]
RequiredForOnline=no
[Network]
ConfigureWithoutCarrier=true
LinkLocalAddressing=no

```

**`31-br-lan-bind.network`** (LAN 口加入网桥)

```ini
[Match]
Name=lan*
[Link]
RequiredForOnline=no
[Network]
Bridge=br-lan
ConfigureWithoutCarrier=true

```

### 2.4 核心网关配置 (IPv6 修复关键)

**`32-br-lan-bridge.network`**
这是最重要的网络配置文件，包含了 Android 断流修复和 ULA 地址配置。

```ini
[Match]
Name=br-lan

[Network]
Address=192.168.0.1/24
DHCPServer=yes
# 开启 IPv6 PD (前缀代理)
DHCPPrefixDelegation=yes
IPv6AcceptRA=no
IPv6SendRA=yes
# 【关键】添加一个静态 IPv6 内网(ULA)地址，用于稳定的 DNS 监听
Address=fd00::1/64

[DHCPServer]
PoolOffset=2
PoolSize=150
EmitDNS=yes
DNS=192.168.0.1
# 【优化】延长 IPv4 租期
DefaultLeaseTimeSec=12h
MaxLeaseTimeSec=24h

[IPv6SendRA]
EmitDNS=yes
# 【关键】下发 ULA 地址作为 DNS
DNS=fd00::1

# 【关键：Android 断流修复】
# 1. 缩短 RA 发送间隔
MinTransmitRouterAdvertisementIntervalSec=3
MaxTransmitRouterAdvertisementIntervalSec=10
# 2. 大幅延长路由有效期(2小时)
RouterLifetimeSec=7200
# 3. 提高路由优先级
RouterPreference=high
EmitDomains=no

[DHCPPrefixDelegation]
UplinkInterface=ppp0
SubnetId=0x10

```

---

## 第三步：PPPoE 拨号与 DNS

### 3.1 拨号配置

安装 `ppp` 包。编辑 `/etc/ppp/peers/private`：

```
plugin pppoe.so
wan0
name "你的宽带账号"
persist
defaultroute
hide-password
noauth

```

编辑密码文件 `/etc/ppp/pap-secrets`：

```text
"你的宽带账号" * "你的宽带密码"

```

### 3.2 强化 PPPoE 服务稳定性

创建覆盖配置，确保无限重拨。
`mkdir -p /etc/systemd/system/ppp@.service.d/`
创建 `/etc/systemd/system/ppp@.service.d/override.conf`：

```ini
[Service]
Restart=on-failure
RestartSec=5s

```

### 3.3 Networkd 对接 PPPoE

让 systemd-networkd 接管 ppp0 接口。
**`/etc/systemd/network/33-ppp.network`**

```ini
[Match]
Name=ppp0
[Link]
RequiredForOnline=no
[Network]
ConfigureWithoutCarrier=true
DefaultRouteOnDevice=true
LinkLocalAddressing=ipv6
IPv6AcceptRA=yes
DHCP=ipv6
[DHCPv6]
PrefixDelegationHint=::/64
WithoutRA=solicit
UseDNS=no
[IPv6AcceptRA]
UseDNS=no

```

### 3.4 DNS 配置 (Systemd-resolved)

**`/etc/systemd/resolved.conf.d/01-custom.conf`**

```ini
[Resolve]
DNS=223.5.5.5 2400:3200::1
FallbackDNS=8.8.8.8 2001:4860:4860::8888
DNSStubListener=no
# 监听 LAN IP 和 ULA IP
DNSStubListenerExtra=192.168.0.1
DNSStubListenerExtra=fd00::1

```

---

## 第四步：配置 IPv6 DDNS (动态域名解析)

由于拨号上网的 IPv6 地址是动态变化的，我们需要配置 DDNS 以便通过域名访问家里的服务。我们将使用 `ddclient` 配合 Cloudflare API，并利用 PPPoE 的钩子脚本实现“拨号即更新”。

### 4.1 提取公网 IPv6 脚本

由于网卡上可能存在 Link-Local (fe80) 或 ULA (fd00) 地址，我们需要一个脚本来精准提取公网 Global IPv6 地址。

创建文件 **`/usr/local/bin/get-ip`**：

```
#!/bin/bash
# 提取 ppp0 接口的 IPv6 地址，过滤掉本地回环和链路本地地址
/sbin/ip -6 addr show dev ppp0 | grep inet6 | awk -F '[ \t]+|/' '{print $3}' | grep -v ^::1 | grep -v ^fe80

```

赋予执行权限：

```
sudo chmod +x /usr/local/bin/get-ip

```

### 4.2 配置 DDClient (以 Cloudflare 为例)

安装 `ddclient` 包，然后编辑配置文件 **`/etc/ddclient/ddclient.conf`**：

```ini
daemon=1800              # 每 300 秒(5分钟)检查一次 (作为守护进程运行时)
syslog=yes               # 记录日志
pid=/var/run/ddclient.pid
ssl=yes                  # 使用 SSL 连接
usev4=disabled           # 禁用 IPv4 (家庭宽带通常是大内网)
# 关键：使用 cmdv6 调用刚才的脚本获取 IP
usev6=cmdv6, cmdv6='/usr/local/bin/get-ip'

# Cloudflare 配置示例
protocol=cloudflare,        \
zone=yourdomain.com,        \
ttl=1,                      \
login=token,                \
password='你的_Cloudflare_API_Token', \
sub.yourdomain.com

```

### 4.3 配置 PPPoE 拨号触发钩子

为了实现 IP 变动后立即更新，而不是等待 `ddclient` 的轮询周期，我们利用 PPP 的 `ipv6-up.d` 钩子。

创建文件 **`/etc/ppp/ipv6-up.d/01-ddns.sh`**：

```
#!/bin/sh
# 拨号成功后等待 5 秒（确保网络稳定），然后强制运行一次 ddclient
sleep 5
ddclient -daemon=0 -syslog -verbose -noquiet > /var/log/ddns.log

```

赋予执行权限：

```
sudo chmod +x /etc/ppp/ipv6-up.d/01-ddns.sh

```

这样配置后，每次路由重启或重新拨号，DDNS 都会在几秒内自动更新。

---

## 第五步 (可选分支)：使用 Dae 实现内核级高性能代理

**如果你追求极致的性能和更简洁的配置，可以选择 Dae 替代 V2Ray + TProxy 方案。**

**Dae 的优势：**

* **内核级分流 (eBPF)**：直接在内核层处理流量，CPU 占用极低，性能吊打传统用户态代理。
* **配置简单**：单个配置文件搞定分流、DNS 和节点，无需维护复杂的 iptables/nftables 脚本。
* **生态友好**：完美支持 Arch Linux（内核版本新）。

**注意：** 如果选择此方案，请**跳过**后文的 **第五步 (部署 V2Ray)** 和 **第七步 (透明代理注入)**。第六步的基础防火墙（NAT/安全过滤）依然建议保留。

### 5.1 安装 Dae

Dae 依赖 eBPF，对内核版本要求较高（Arch Linux 默认内核已满足）。为了获得极致性能，建议根据 CPU 指令集选择合适的版本。

**1. 检查 AVX2 指令集支持**

**Intel N100** 处理器是支持 **AVX2** 指令集的，这能显著提升加解密效率和数据包处理性能。建议先确认一下：

```
# 如果输出 avx2，则说明支持
grep -o 'avx2' /proc/cpuinfo | head -n1

```

**2. 选择版本安装**

* **方案 A：常规版 (兼容性好)**

如果你不确定或通过上述命令没有看到 avx2，使用标准版。

```
yay -S dae-bin

```

* **方案 B：AVX2 优化版 (N100 推荐)**

如果确认支持 AVX2，推荐安装优化版（通常对应 `x86_64_v3` 架构）。这能榨干 N100 的每一滴性能。
*(注：如果 AUR 中搜索不到 `dae-avx2-bin`，可以选择安装 `dae` 源码包进行本地编译，Go 编译器会自动适配本机架构)*

```
# 尝试安装预编译的 AVX2 版本
yay -S dae-avx2-bin

# 或者：从源码编译 (自动适配 AVX2)
# yay -S dae

```


### 5.2 编写配置文件

创建或编辑 `/etc/dae/config.dae`。

```perl
global {
    # 绑定 LAN 接口 (网桥)
    lan_interface: br-lan
    
    # 绑定 WAN 接口 (自动识别，通常是 ppp0 或 wan0)
    wan_interface: auto

    log_level: info
    allow_insecure: false
    
    # 自动优化内核参数 (如 ip_forward)
    auto_config_kernel_parameter: true
}

node {
    # 在此处填入你的节点链接
    # 支持 vmess, vless, trojan, hysteria2, tuic 等主流协议
    proxy1: 'vmess://......'
}

dns {
    upstream {
        # 上游 DNS 定义
        googledns: 'tcp+udp://dns.google:53'
        ali_quic: 'quic://dns.alidns.com:853'
    }

    routing {
        request {
            # 国内域名和特定 Geosite 使用阿里 DNS (QUIC)
            # 注意：混合 Geosite 和域名时，普通域名必须加 suffix: 前缀
            qname(
                suffix: fiepi.com,
                suffix: jetbrains.com.cn,
                suffix: mirrorbits.lineageos.org,
                suffix: cn.bing.com,
                suffix: wenku8.net,
                geosite: private,
                geosite: xiaomi,
                geosite: category-games,
                geosite: apple,
                geosite: cn
            ) -> ali_quic
            
            # 默认使用 Google DNS
            fallback: googledns
        }
    }
}

group {
    proxy {
        # 策略组：选择延迟最低的节点
        policy: min_moving_avg
    }
}

routing {
    # === 1. 强制直连规则 ===
    # 关键系统进程直连
    pname(NetworkManager, systemd-resolved, dnsmasq) -> must_direct
    
    # 保留地址、组播地址直连
    dip(224.0.0.0/3, 'fd00::/8', 'ff00::/8') -> direct
    dip(geoip:private) -> direct
    dip(geoip:cn) -> direct
    
    # 广告屏蔽
    domain(geosite: category-ads-all) -> block

    # 直连域名列表 (对应 DNS 规则)
    domain(
        fiepi.com,
        jetbrains.com.cn,
        mirrorbits.lineageos.org,
        cn.bing.com,
        wenku8.net,
        geosite:xiaomi,
        geosite:private,
        geosite:category-vpnservices,
        geosite:category-games,
        geosite:apple,
        geosite:cn
    ) -> direct

    # === 2. 强制代理规则 ===
    domain(
        bangumi.moe,
        flathub.org,
        community.oneplus.com,
        gradle.org,
        geosite:wikimedia,
        geosite:mihoyo,
        geosite:github,
        geosite:openai
    ) -> proxy

    # === 3. 端口分流 (关键) ===
    # 仅代理常用端口 (1-1024)，其余高位端口直连
    # 这能有效解决 BT 下载跑流量、游戏 UDP 语音异常等问题
    dport(1-1024) -> proxy

    # === 4. 兜底规则 ===
    # 上面没匹配到的全部直连
    fallback: direct
}

```

### 5.3 启动 Dae

```
# 启用并立即启动
systemctl enable --now dae.service

```

---


## 第五步：部署 V2Ray (透明代理后端)

*(如果你选择了 Dae 方案，请跳过此步)*

我们将采用“配置分离”的方式，便于管理。

### 5.1 配置文件结构

创建目录：`mkdir -p /etc/v2ray/confs`

 **1. 基础配置 `/etc/v2ray/confs/01_base.json`**
(包含 Inbounds 和 Routing)

```json
{
  "log": { "loglevel": "warning" },
  "inbounds": [
  {
      "tag": "transparent",
      "listen": "::",
      "port": 12345,
      "protocol": "dokodemo-door",
      "settings": {
        "network": "tcp,udp",
        "followRedirect": true
      },
      "sniffing": {
        "enabled": true,
        "excludedDomain": [],
        "destOverride": [
          "http",
          "tls"
        ]
      },
      "streamSettings": {
        "sockopt": {
          "tproxy": "tproxy",
          "mark": 255
        }
      }
    },
    {
      "tag": "socks", "port": 1080, "listen": "192.168.0.1", "protocol": "socks",
      "settings": { "auth": "noauth" }
    }
  ],
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      { "type": "field", "port": 53, "network": "udp", "outboundTag": "dns-out" },
      { "type": "field", "ip": ["geoip:cn", "geoip:private"], "outboundTag": "direct" },
      { "type": "field", "domain": ["geosite:cn"], "outboundTag": "direct" }
    ]
  }
}

```

**2. 节点配置 `/etc/v2ray/confs/02_outbound.json`**
(包含 Outbounds)

```
{
  "outbounds": [
    {
      "tag": "proxy",
      "protocol": "vmess", 
      "settings": { ...你的节点配置... },
      "streamSettings": { "sockopt": { "mark": 255 } } 
    },
    { "tag": "direct", "protocol": "freedom", "streamSettings": { "sockopt": { "mark": 255 } } },
    { "tag": "dns-out", "protocol": "dns", "streamSettings": { "sockopt": { "mark": 255 } } }
  ]
}

```

*注意：所有 Outbounds 必须加上 `sockopt: { mark: 255 }`，防止流量回环！*

### 5.2 V2Ray Systemd 服务

告诉 Systemd 读取 `/etc/v2ray/confs` 目录。

**`/etc/systemd/system/v2ray-tproxy.service`**

```ini
[Unit]
Description=V2Ray Service (Multi-conf)
Documentation=https://www.v2fly.org/
After=network.target nss-lookup.target

[Service]
User=root
# 给予网络权限
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
NoNewPrivileges=true
# 关键：指定配置目录
ExecStart=/usr/bin/v2ray run -confdir /etc/v2ray/confs
Restart=on-failure
RestartPreventExitStatus=23
LimitNPROC=512
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target

```

---

## 第六步：现代防火墙 Nftables (基础底座)

**无论选择 V2Ray 还是 Dae，这一步都建议保留。**
它负责基础的 NAT (让局域网能上网) 和端口安全。
*注意：Dae 自带 Flow Offloading 效果，本配置中的 offload 不会冲突，可共存。*

基础防火墙负责 NAT、安全过滤和硬件加速。即使不翻墙，这层也必须运行。

**`/etc/nftables.conf`**

```nft
#!/usr/bin/nft -f
flush ruleset

table inet router {
    set lan_nets { type ipv4_addr; flags interval; elements = { 192.168.0.0/24 }; }
    set open_tcp_ports { type inet_service; flags interval; elements = { 22, 1080 }; }
    
    # 开启 Flow Offloading (性能核心)
    flowtable f { hook ingress priority 0; devices = { wan0, ppp0, br-lan }; }

    chain forward {
        type filter hook forward priority filter; policy drop;
        ip protocol { tcp, udp } flow offload @f
        tcp flags syn tcp option maxseg size set rt mtu
        meta l4proto { icmp, icmpv6 } accept
        ct state established,related accept
        iifname "br-lan" accept
    }

    chain input {
        type filter hook input priority filter; policy drop;
        iifname "lo" accept
        ct state established,related accept
        iifname "br-lan" accept
        tcp dport @open_tcp_ports accept
        udp dport 546 accept
        meta l4proto { icmp, icmpv6 } accept
    }

    chain dstnat {
        type nat hook prerouting priority dstnat; policy accept;
    }

    chain srcnat {
        type nat hook postrouting priority srcnat; policy accept;
        oifname { "wan0", "ppp0" } masquerade
    }
}

```

---

## 第七步：透明代理注入 (可插拔设计)

*(如果你选择了 Dae 方案，请**跳过**此步，Dae 会自动处理流量劫持)*

我们将代理规则与基础防火墙解耦。需要三个文件。

### 7.1 注入规则

**`/etc/nftables/tproxy.nft`**

```nft
table inet router {
    # === 变量定义 ===
    # IPv4 保留
    set tproxy_reserved {
        type ipv4_addr; flags interval
        elements = { 127.0.0.0/8, 224.0.0.0/4, 255.255.255.255 }
    }
    set lan_nets {
        type ipv4_addr; flags interval
        elements = { 192.168.0.0/24 }
    }

    # IPv6 保留 (包含回环、多播、链路本地)
    set tproxy_reserved6 {
        type ipv6_addr; flags interval
        elements = { ::1/128, ff00::/8, fe80::/10 }
    }
    # IPv6 局域网段 (ULA)
    set lan_nets6 {
        type ipv6_addr; flags interval
        elements = { fd00::/8 }
    }

    # === 处理局域网流量 (TProxy) ===
    chain tproxy_logic {
        # 1. 强制 DNS 劫持 (关键！)
        # 无论 IPv4 还是 IPv6 DNS 查询，强制转发给 V2Ray
        meta l4proto { tcp, udp } th dport 53 meta mark set 1 tproxy ip to 127.0.0.1:12345 accept
        meta l4proto { tcp, udp } th dport 53 meta mark set 1 tproxy ip6 to [::1]:12345 accept

        # 2. 直连保留地址和局域网
        ip daddr @tproxy_reserved return
        ip6 daddr @tproxy_reserved6 return
        ip daddr @lan_nets return
        ip6 daddr @lan_nets6 return

        # 3. 防止回环标记
        meta mark 0xff return

        # 4. 全局代理 (IPv4 -> 127.0.0.1, IPv6 -> [::1])
        meta l4proto { tcp, udp } meta mark set 1 tproxy ip to 127.0.0.1:12345 accept
        meta l4proto { tcp, udp } meta mark set 1 tproxy ip6 to [::1]:12345 accept
    }

    # === 处理本机流量 (Output) ===
    chain output_mask {
        # 1. 直连保留地址
        ip daddr @tproxy_reserved return
        ip6 daddr @tproxy_reserved6 return

        # 2. 直连局域网 (关键修复！)
        # 必须确保路由器能直连访问网关 IP、手机 IP 等，否则 SSH 回包和 DNS 转发会断
        ip daddr @lan_nets return
        ip6 daddr @lan_nets6 return

        # 3. 防止 V2Ray 自身死循环 (Mark 255)
        meta mark 0xff return

        # 4. 代理本机剩余流量
        # 这会让路由器本机也具备翻墙能力 (如 pacman 下载)
        meta l4proto { tcp, udp } meta mark set 1 accept
    }

    # === 挂载点 ===
    chain mangle_prerouting {
        type filter hook prerouting priority mangle; policy accept;
        meta l4proto tcp socket transparent 1 meta mark set 1 accept
        iifname "br-lan" jump tproxy_logic
    }

    chain mangle_output {
        type route hook output priority mangle; policy accept;
        jump output_mask
    }
}

```

### 7.2 清理规则

**`/etc/nftables/tproxy-remove.nft`**

```nft
#!/usr/bin/nft -f
delete chain inet router mangle_prerouting
delete chain inet router mangle_output
delete chain inet router tproxy_logic
delete chain inet router output_mask
delete set inet router tproxy_reserved
delete set inet router tproxy_reserved6
delete set inet router lan_nets
delete set inet router lan_nets6

```

### 7.3 代理管理服务

**`/etc/systemd/system/tproxy.service`**

```ini
[Unit]
Description=Transparent Proxy Rules (Nftables + IP Rule)
After=network.target v2ray-tproxy.service
Wants=v2ray-tproxy.service

[Service]
Type=oneshot
RemainAfterExit=yes

# --- 启动前清理 (双栈) ---
ExecStartPre=-/usr/bin/ip route del local 0.0.0.0/0 dev lo table 100
ExecStartPre=-/usr/bin/ip rule del fwmark 1 table 100
ExecStartPre=-/usr/bin/ip -6 route del local ::/0 dev lo table 100
ExecStartPre=-/usr/bin/ip -6 rule del fwmark 1 table 100
ExecStartPre=-/usr/bin/nft -f /etc/nftables/tproxy-remove.nft

# --- 启动逻辑 (关键：双栈策略) ---
# IPv4
ExecStart=/usr/bin/ip rule add fwmark 1 table 100
ExecStart=/usr/bin/ip route add local 0.0.0.0/0 dev lo table 100
# IPv6
ExecStart=/usr/bin/ip -6 rule add fwmark 1 table 100
ExecStart=/usr/bin/ip -6 route add local ::/0 dev lo table 100
# 注入防火墙
ExecStart=/usr/bin/nft -f /etc/nftables/tproxy.nft

# --- 停止逻辑 ---
ExecStop=-/usr/bin/nft -f /etc/nftables/tproxy-remove.nft
ExecStop=-/usr/bin/ip route del local 0.0.0.0/0 dev lo table 100
ExecStop=-/usr/bin/ip rule del fwmark 1 table 100
ExecStop=-/usr/bin/ip -6 route del local ::/0 dev lo table 100
ExecStop=-/usr/bin/ip -6 rule del fwmark 1 table 100

[Install]
WantedBy=multi-user.target

```

---

## 第八步：启动与验证

根据你选择的方案，启动服务略有不同。

### 方案 A：Dae (推荐)

1. **加载配置**：`systemctl daemon-reload`
2. **基础服务**：`systemctl enable --now systemd-networkd systemd-resolved ppp@private`
3. **基础防火墙**：`systemctl enable --now nftables`
4. **核心代理**：`systemctl enable --now dae`

### 方案 B：V2Ray + TProxy

1. **加载配置**：`systemctl daemon-reload`
2. **基础服务**：`systemctl enable --now systemd-networkd systemd-resolved ppp@private`
3. **基础防火墙**：`systemctl enable --now nftables`
4. **V2Ray后端**：`systemctl enable --now v2ray-tproxy`
5. **TProxy注入**：`systemctl enable --now tproxy`

### 验证方法

* **查看内核日志 (Dae)**：运行 `journalctl -u dae -f`，观察是否有流量日志。
* **Android 测试**：连接 WiFi，锁屏放置 1 小时，解锁后立即打开网页，应无延迟。
* **性能测试**：在局域网内跑满 2.5G 带宽，使用 `htop` 查看软路由 CPU 占用。Dae 方案下 `softirq` 应非常低。

---

至此，一台配置了企业级防火墙、硬件加速和无感代理的高性能软路由就搭建完成了。享受你的 2.5G 网络吧！
