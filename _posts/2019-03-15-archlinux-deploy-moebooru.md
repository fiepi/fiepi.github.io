---
layout: post
title: 记录 ArchLinux 下部署 Moebooru 的过程
description: 记录 ArchLinux 下部署 Moebooru 的过程
categories: [archlinux]
keywords: archlinux, moebooru, booru
---

[Moebooru](https://github.com/moebooru/moebooru) 是一个基于 Danbooru1 修改的贴图版引擎， 比较有名的网站 konachan.com 和 yande.re 使用的正是该引擎。

## 需要的依赖包
{% highlight bash %}
git nodejs imagemagick jhead libxslt libyaml readline pcre openssl postgresql nginx ruby rubygems
{% endhighlight %}

## 配置数据库
新增一个用户名为 moebooru 的用户
{% highlight bash %}
sudo useradd -m -d /home/moebooru moebooru
{% endhighlight %}
对数据库进行初始化配置
{% highlight bash %}
sudo -iu postgres
# 数据库数据路径
initdb -D /var/lib/postgres/data
exit
# systemd 管理进程
sudo systemctl start postgresql.service
sudo systemctl status postgresql.service
sudo systemctl enable postgresql.service

sudo -iu postgres
[postgres@archlinux ~]$ createuser --interactive
Enter name of role to add: moebooru
Shall the new role be a superuser? (y/n) y
# 修改用户密码
[postgres@archlinux ~]$ psql
postgres=# ALTER USER moebooru WITH PASSWORD 'password'
postgres=# \q
# 创建数据库
[postgres@archlinux ~]$ createdb moebooru
[postgres@archlinux ~]$ exit
{% endhighlight %}

## 安装 Moebooru

{% highlight bash %}
sudo -iu moebooru
git clone https://github.com/moebooru/moebooru.git ~/live
cd live
mkdir -p public/data/{avatars,frame,frame-preview,image,inline,jpeg,preview,sample,search}
cp config/database.yml.example config/database.yml
cp config/local_config.rb.example config/local_config.rb
gem install bundler:1.17.2
# 安装缺失的依赖
bundle install --path ~/live/vendor/bundle
# 生成密钥
bundle exec rake secret
{% endhighlight %}
config/local_config.rb 比较关键的配置
{% highlight bash %}
CONFIG["secret_key_base"] = "生成的密钥"
# Servers for static files (assets and uploaded files)
CONFIG[:file_hosts] = { :files => CONFIG["server_host"], :assets => CONFIG["server_host"] }
{% endhighlight %}

config/database.yml 数据库配置，填上数据库信用户息

{% highlight bash %}
    username: moebooru
    password: password
    host: localhost
{% endhighlight %}

config/init_config.rb
{% highlight bash %}
# This is a salt used to make dictionary attacks on account passwords harder.
CONFIG["password_salt"] = "your-hash-salt"
# https 需要开启
# Set secure to false by default due to ssl requirement
CONFIG["secure"] = true
# 开启之后 nginx 配置需要进行相应修改
CONFIG["use_pretty_image_urls"] = true
CONFIG["web_server"] = "nginx"
{% endhighlight %}
其余配置根据需要修改

初始化
{% highlight bash %}
bundle exec rake db:create
bundle exec rake db:reset
bundle exec rake db:migrate
bundle exec rake i18n:js:export
bundle exec rake assets:precompile
{% endhighlight %}

## Nginx 配置
用的是 [Let's Encrypt](https://letsencrypt.org) 的免费证书
{% highlight bash %}
server { 
	root /home/moebooru/live/public; 
	listen 443 ssl http2;
	
	ssl_certificate /etc/letsencrypt/live/moe.fiepi.com/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/moe.fiepi.com/privkey.pem;
	include /etc/letsencrypt/options-ssl-nginx.conf;
	ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

	resolver 8.8.8.8 8.8.4.4 valid=300s;
	resolver_timeout 5s;

	server_name moe.fiepi.com;

	location / { 
		try_files /cache/$uri /cache/$uri.html $uri @moe; 
	} 
	location @moe {
        	expires off; proxy_pass http://127.0.0.1:8080; 
        	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
        	proxy_set_header X-Real-IP $remote_addr; proxy_redirect off; 
        	proxy_set_header Host $host:$server_port; proxy_set_header X-Forwarded-Proto $scheme;
		rewrite "^/image/([0-9a-f]{2})([0-9a-f]{2})([0-9a-f]{28})(/.*)?(\.[a-z]*)" "/data/image/$1/$2/$1$2$3$5";
		rewrite "^/sample/([0-9a-f]{2})([0-9a-f]{2})([0-9a-f]{28})(/.*)?(\.[a-z]*)" "/data/sample/$1/$2/$1$2$3$5";
    	}
}

server {
	listen 80;
        server_name moe.fiepi.com;
	rewrite ^/(.*) https://moe.fiepi.com/$1 permanent;
}
{% endhighlight %}

## 添加 systemd 配置

{% highlight bash %}
# /etc/systemd/system/moebooru.service 
[Unit]
Description=Moebooru
After=syslog.target
After=network.target

[Service]
Type=simple
User=moebooru
WorkingDirectory=/home/moebooru/live
ExecStart=/home/moebooru/.gem/ruby/2.6.0/bin/bundle exec unicorn
Restart=always

[Install]
WantedBy=multi-user.target
{% endhighlight %}

最后启动 moebooru.service 就 OK 了。

参考： [Moebooru部署笔记](https://ioover.net/dev/moebooru-deploy-note)