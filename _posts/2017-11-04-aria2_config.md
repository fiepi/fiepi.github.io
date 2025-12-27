---
layout: post
title: 记录 archlinux 下 aria2+https 的配置过程
categories: [archlinux]
description: 记录 archlinux 下 aria2+https 的配置过程
keywords: archlinux, aria2, https, letsencrypt
---

Aria2 是一个轻量的多协议、多线程下载器。这里记录一下安装配置过程。

<!-- more -->

# 1. 安装 

{% highlight bash %}
sudo pacman -S aria2 certbot nginx
{% endhighlight %}

# 2. 配置

假设下载服务器域名为：aria2.fiepi.com

## #  配置临时 web 服务器获取免费 Let's Encrypt 证书

{% highlight bash %}
sudo vim /etc/nginx/nginx.conf
# 添加到 http 内
include vhost/*.conf;
{% endhighlight %}
{% highlight bash %}
# 创建 nginx 配置文件
sudo mkdir /etc/nginx/vhost
sudo vim /etc/nginx/vhost/aria2.fiepi.com.conf
{% endhighlight %}

{% highlight conf %}
server {
    listen 80;
    server_name aria2.fiepi.com;
    index index.html index.htm;
    root /home/fiepi/web/root;
    autoindex on;
    autoindex_exact_size off;
    autoindex_localtime on;
    location / {
    }
    location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|flv|ico)$ {
        expires 30d;
        access_log off;
    }
    location ~ .*\.(js|css)?$ {
        expires 7d;
        access_log off;
    }
}
{% endhighlight %}
{% highlight bash %}
# 测试配置无误后重启 nginx 并生成证书
sudo nginx -t
sudo systemctl restart nginx.service
sudo certbot certonly --webroot -w /home/fiepi/web/root -d aria2.fiepi.com
{% endhighlight %}
证书路径：/etc/letsencrypt/archive/aria2.fiepi.com
{% highlight bash %}
 # 修改权限让 aria2 能够读取
sudo chown -R fiepi:users /etc/letsencrypt/archive

#移除临时 web 配置
sudo mv /etc/nginx/vhost/aria2.fiepi.com.conf /etc/nginx/vhost/aria2.fiepi.com.conf.bak
sudo systemctl restart nginx.service
{% endhighlight %}

## #  创建 aria2 配置

{% highlight bash %}
# 随机生成身份认证的 Token：
openssl rand -hex 15
# 路径
mkdir -p ~/.config/aria2
vim ~/.config/aria2/aria2.conf
{% endhighlight %}

{% highlight conf %}
## 下载路径
dir=~/Downloads/aria2
## 生成随机 Token: openssl rand -hex 15
rpc-secret=xxxxxx
rpc-secure=true
## 证书
rpc-certificate=/etc/letsencrypt/archive/aria2.fiepi.com/fullchain1.pem
rpc-private-key=/etc/letsencrypt/archive/aria2.fiepi.com/privkey1.pem
## http代理
# http-proxy=http://127.0.0.1:8118
## 上传速度限制 0 为不限制
max-overall-upload-limit=10K
max-upload-limit=2K
## 是否预先分配磁盘空间
file-allocation=prealloc
## 是否继续下载未完成的文件
continue=true
## 日志级别，可以为debug, info, notice, warn 或 error
log-level=info
## 每下载任务最大连接数
max-connection-per-server=10
## 下载进度输出的间隔时间
#summary-interval=120
## 是否以进程的方式启动
daemon=true
## 是否启用rpc
enable-rpc=true
## rpc监听端口
rpc-listen-port=6800
## 是否在所有网卡上启用监听
rpc-listen-all=true
## 最大同时下载任务数，根据实际情况修改
#max-concurrent-downloads=3
## 会话保存文件，进程退出时保存未下载完成的会话
save-session=/home/fiepi/.config/aria2/session.lock
## 启动输入文件，进程启动时读取上次未下载完成的会话
input-file=/home/fiepi/.config/aria2/session.lock
## 日志文件，根据实际情况修改
#log=~/.config/aria2/aria.log
## 是否关闭ipv6
disable-ipv6=true
## 磁盘缓存
disk-cache=0
## 超时时间
#timeout=600
## 重试等待时间
#retry-wait=30
## 最大重试次数，0代表可以无限次重试
#max-tries=0
## user agent，此处所填值用于伪装成百度云网盘客户端
user-agent=netdisk;4.4.0.6;PC;PC-Windows;6.2.9200 WindowsBaiduYunGuanJia
{% endhighlight %}

## #  创建 systemd 守护进程

{% highlight bash %}
sudo vim /etc/systemd/user/aria2.service
{% endhighlight %}
{% highlight conf %}
[Unit]
Description=Aria2 Service
After=network.target

[Service]
Type=forking
WorkingDirectory=%h
ExecStart=/usr/bin/aria2c --daemon --enable-rpc --rpc-listen-all --rpc-allow-origin-all -c -D  --conf-path=%h/.config/aria2/aria2.conf

[Install]
WantedBy=default.target

{% endhighlight %}

{% highlight bash %}
# 启动
systemctl --user start aria2.service
systemctl --user enable aria2.service
{% endhighlight %}

过去用户进程在用户登出时会被杀死，现在 Arch 已经修改了默认编译参数，所以不需要额外的设置。详见  [Arch Wiki Systemd/User](https://wiki.archlinux.org/index.php/Systemd/User) 的 [Kill user processes on logout](https://wiki.archlinux.org/index.php/Systemd/User#Kill_user_processes_on_logout) 部分。

## #  防火墙配置
{% highlight bash %}
# 打开 6800 端口
sudo vim /etc/iptables/iptables.rules
# 添加
-A INPUT -p tcp -m tcp --dport 6800 -j ACCEPT

#生效
sudo iptables-restore < /etc/iptables/iptables.rules
{% endhighlight %}

最后打开 Yaaw（[yaaw.fiepi.com](https://yaaw.fiepi.com)） 前端，填入配置：

````
https://token:xxxxxx@aria2.fiepi.com:6800/jsonrpc
````
