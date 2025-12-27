---
layout: post
title: ss-redir+iptables+ipset+dnsmasq 配置记录
description: ss-redir+iptables+ipset+dnsmasq 配置记录
categories: [archlinux]
keywords: ss-redir, iptables, archlinux, ipset, dnsmasq
---

记录 ss-redir 透明代理配置过程。
<!-- more -->

## #  相关软件包

{% highlight bash %}
sudo pacman -S dnsmasq ipset shadowsocks-libev
{% endhighlight %}

## #  ss-redir 配置

{% highlight bash %}
# 创建配置文件
sudo vim /etc/shadowsocks/ss-redir.json
{% endhighlight %}
{% highlight json %}
{
	"server": "[server_ip]",
	"server_port": [server_port],
	"local_port": 10800,
	"local_address": "127.0.0.1",
	"password": "password",
	"timeout": 300,
	"method": "aes-256-gcm"
}
{% endhighlight %}
{% highlight bash %}
# systemd 守护进程
sudo vim /etc/systemd/system/ss-redir.service
{% endhighlight %}
{% highlight conf %}
[Unit]
Description=Shadowsocks-Libev Client Service Redir Mode
After=network.target

[Service]
Type=simple
User=nobody
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
ExecStart=/usr/bin/ss-redir -c /etc/shadowsocks/ss-redir.json -u

[Install]
WantedBy=multi-user.target
{% endhighlight%}

{% highlight bash %}
# 保持自启动
sudo systemctl start ss-redir.service
sudo systemctl enable ss-redir.service
{% endhighlight %}

## #  iptables + ipset 实现 chnroute 分流

{% highlight bash %}
# 获取中国 IP
su
wget -O- 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > /etc/chnroute.txt
exit
{% endhighlight %}
{% highlight bash %}
# 停用默认防火墙自启动服务
sudo systemctl disable iptables.service
# 新建启动脚本
sudo vim /etc/iptables/ss-up.sh
{% endhighlight %}
{% highlight bash %}
#!/bin/bash

ipset -N chnroute hash:net maxelem 65536

for ip in $(cat '/etc/chnroute.txt'); do
  ipset add chnroute $ip
done

iptables -t nat -N SHADOWSOCKS

# 直连服务器 IP
iptables -t nat -A SHADOWSOCKS -d [server_ip]/24 -j RETURN

# 允许连接保留地址
iptables -t nat -A SHADOWSOCKS -d 0.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 10.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 127.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 169.254.0.0/16 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 172.16.0.0/12 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 192.168.0.0/16 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 224.0.0.0/4 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 240.0.0.0/4 -j RETURN

# 直连中国 IP
iptables -t nat -A SHADOWSOCKS -p tcp -m set --match-set chnroute dst -j RETURN
iptables -t nat -A SHADOWSOCKS -p icmp -m set --match-set chnroute dst -j RETURN

# 重定向到 ss-redir 端口
iptables -t nat -A SHADOWSOCKS -p tcp -j REDIRECT --to-port 10800
iptables -t nat -A SHADOWSOCKS -p udp -j REDIRECT --to-port 10800
iptables -t nat -A OUTPUT -p tcp -j SHADOWSOCKS
{% endhighlight %}
{% highlight bash %}
#测试执行情况
sudo chmod +x /etc/iptables/ss-up.sh
sudo sh /etc/iptables/ss-up.sh
{% endhighlight %}
{% highlight bash %}
# 创建停止脚本
sudo vim /etc/iptables/ss-down.sh
sudo chmod +x /etc/iptables/ss-down.sh
{% endhighlight %}
{% highlight bash %}
#!/bin/bash
iptables -t nat -D OUTPUT -p icmp -j SHADOWSOCKS
iptables -t nat -D OUTPUT -p tcp -j SHADOWSOCKS
iptables -t nat -F SHADOWSOCKS
iptables -t nat -X SHADOWSOCKS
ipset destroy chnroute
{% endhighlight %}

{% highlight bash %}
# 复制修改原 iptables 的 systemd 配置
sudo cp /usr/lib/systemd/system/iptables.service /etc/systemd/system/iptables-proxy.service
sudo vim /etc/systemd/system/iptables-proxy.service
{% endhighlight %}
{% highlight conf %}
[Unit]
Description=Packet Filtering Framework and Shadowsocks-chnroute
Before=network-pre.target
Wants=network-pre.target

[Service]
Type=oneshot
ExecStart=/etc/iptables/ss-up.sh
ExecStop=/etc/iptables/ss-down.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target

{% endhighlight %}

{% highlight bash %}
# 检查并添加自启动
sudo systemctl daemon-reload
sudo systemctl start iptables-proxy.service
sudo systemctl status iptables-proxy.service
sudo systemctl enable iptables-proxy.service
{% endhighlight %}

## #  dnsmasq 的配置
Dnsmasq 提供 DNS 缓存和 DHCP 服务功能，通过缓存 DNS 请求来提高对访问过的网址的连接速度。
{% highlight bash %}
sudo vim /etc/dnsmasq.conf
{% endhighlight %}
{% highlight conf %}
# 去掉这几行的注释
resolv-file=/etc/resolv.dnsmasq.conf
listen-address=127.0.0.1
{% endhighlight %}

{% highlight bash %}
# 修改 Dnsmasq DNS
sudo vim /etc/resolv.dnsmasq.conf 
{% endhighlight %}
{% highlight conf %}
nameserver 8.8.8.8
nameserver 8.8.4.4
{% endhighlight %}
{% highlight bash %}
# 修改系统 DNS
sudo vim /etc/resolv.conf
{% endhighlight%}
{% highlight conf %}
nameserver 127.0.0.1
{% endhighlight %}
{% highlight bash %}
#防止被修改
sudo chattr +i /etc/resolv.conf
{% endhighlight %}
更多细节： [ Dnsmasq - Arch Wiki](https://wiki.archlinux.org/index.php/dnsmasq) 

{% highlight bash %}
# 启动 Dnsmasq 
sudo systemctl start dnsmasq.service
sudo systemctl status dnsmasq.service
sudo systemctl enable dnsmasq.service
# 重启系统看看各项功能是否正常
sudo reboot
{% endhighlight %}
