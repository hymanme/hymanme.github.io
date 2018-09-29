---
layout: post
title: Nginx 配置 www 转发以及开启静态文件服务器
summary: 
date: 2018-09-13 00:30:59
categories: Web
tags: [Web, Nginx, Https]
featured-img: swing
---
## www 转发并强制 https 访问

我们经常需要通过省略`www`直接访问主页，比通过 hymane.com 访问 www.hymane.com。其实很简单，server_name 将两个域名全部配置，然后再通过 301 重定向到 https 443端口，或者使用 rewrite 重写到 https 站点，如下：

```bash
server {
     listen 80;
     # 配置多个域名
     server_name www.hymane.com hymane.com;
     access_log  /home/hymane/www/logs/www.hymane.com.log;
     # 将80端口重写至443 SSL 端口
     rewrite ^(.*)$ https://${server_name}$1 permanent;
}

# 443 端口
server {
    listen 443 ssl http2;
    server_name www.hymane.com hymane.com;
    access_log  /home/hymane/www/logs/www.hymane.com.log;
    # root /home/wwwroot;
    ssl on;
    ssl_certificate /etc/nginx/certs/www.hymane.com/ssl_www.crt;
    ssl_certificate_key /etc/nginx/certs/www.hymane.com/ssl_www.key;

    location / {
        proxy_pass http://localhost:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_redirect off;
        #root   /usr/share/nginx/html;
        #index  index.html index.htm;
    }

    location /static/ {
        alias /home/hymane/www/resources/;
        index  index.html index.htm;
    }
}
```

## 想放置一个小型文件服务器

有时候需要放置一些文件在自己的服务器上，比如图片，视频，文档等等，可以解析一个专门用来访问静态文件的 二级域名，如`resource.hymane.com`然后配置 nginx config 文件，首先重写至 443 端口，然后配置 location。

```bash
# resource server
server {
     listen 80;
     server_name resource.hymane.com;
     access_log  /home/hymane/www/logs/resource.hymane.com.log;
     rewrite ^(.*)$ https://${server_name}$1 permanent;
}

server {
    listen 443 ssl http2;
    server_name resource.hymane.com;
    # access_log  /home/hymane/www/logs/resource.hymane.com.log;
    # root /home/hymane/www/resources;
    ssl on;
    ssl_certificate /etc/nginx/certs/www.hymane.com/ssl_resource.crt;
    ssl_certificate_key /etc/nginx/certs/www.hymane.com/ssl_resource.key;

    location / {
    	#务必开启，默认off，自动开始索引，开启之后可以在网页上查看文件目录结构，方便浏览
        autoindex on;
        #默认为on,是否显示文件确切大小
        #改为off后，显示出文件的大概大小，单位是kB或者MB或者GB
        autoindex_exact_size on;
        #默认为off，显示的文件时间为GMT时间。
		#注意:改为on后，显示的文件时间为文件的服务器时间
        autoindex_localtime off;
        #本地目录地址
        root   /home/hymane/www/resources;
        # index  index.html index.htm;
    }
}
```

