---
title: Nginx + SSL 配置实现 https 访问
categories:
- 服务端
tags:
- Nginx
date: 2020-03-19 00:36:46
---

本文将介绍如何通过 Nginx 来配置 SSL，开启接口的 https 调用。如果你正在开发微信小程序，那么可能会帮得上忙。
<!--more-->
### 背景
楼主前阵子写小程序，接口在 Http 环境测试通过，二级域名解析完毕，IP映射完毕，心想万事大吉，发布上线。谁料忘记了小程序生产环境接口必须使用 Https 协议。百密一疏... 看来只能干干运维的活儿了。虽然最后搞定了，不过还是踩了些坑的，记录下。

所以今天要实现功能就是通过我的二级域名 `https://api.tianxiaohu.online` 来访问服务器接口。

### 你需要准备：
这部分不是主要的，所以需要你自行搞定，
> · 域名（楼主域名阿里云买的）
> · 将域名解析到主机IP
> · 服务器安装好 Nginx 环境（安装时务必安装ssl模块 –with-http_ssl_module，很简单，不会的自行Google）

### 为你的域名申请 SSL 证书
为域名申请 SSL 证书其实并不复杂。总所周知，大部分的证书提供商是需要收费的，便宜的也得千把块一年。对于普通个人开发者而言确实是一笔不小的开支。当然了，免费的证书也不是没有，无非就是安全性和稳定性不如收费的。其实平时个人使用还是完全可以 cover 的。
如果你使用的是阿里云，那么在阿里云上就有免费的证书可以[购买](https://common-buy.aliyun.com/?spm=5176.2020520163.c1583915649459.d1583915649459_0.d94056a7EtHbw3.d94056a7EtHbw3&commodityCode=cas#/buy)。购买完，按照阿里云的使用文档，为自己的域名申请证书就ok了，不赘述~

### 配置 Nginx
因为之前你已经域名解析到了你的主机IP，接下来我们需要 Nginx 帮我做的事情就是监听服务器的443端口，然后转发到主机的接口服务上。从而实现通过https访问接口。Nginx具体的安装配置也不过多描述，按照文档来就好，相信不会难倒大家，这里只说一个比较棘手的情况，就是此前已经安装好了Nginx，但是并没有附带安装SSL包。

这种情况是有办法解决的，可以在不卸载 Nginx 的情况下完成SSL包的配置。详细请看[《如何为已经安装好的 Nginx 添加 SSL 模块》]()

开始前先说明下楼主的环境情况（不同环境可能操作有差异）：
> · 服务器：CentOS 7.x
> · 本地机器：MacBook Pro
> · Nginx 版本：1.12.0
> · Nginx 安装目录：/usr/local/nginx

#### 1. 下载 SSL 证书，存放到服务器
如果你的域名 https 已经开通，那么在对应的平台（我的是阿里云）去下载证书文件。在下载的时候要注意选择对应的服务器类型。有 Tomcat、Apache、IIS、Nginx等，本文主要讲解Nginx，所以选择下载Nginx对应的证书文件到你本地。

下载后你会得到两个证书文件，假设为 `cert.pem` 和 `cert.key`。楼主存放到本地 `/Users/Toy/ssl` 目录。

#### 2. 将证书文件推送到服务器
下载好的证书需要放置到服务器上，可以自己找个目录存放。楼主这里放到Nginx的安装目录里 `/usr/local/nginx/cert/ssl`。

创建证书存放目录（目录可自选）：
```javascriptt
cd /usr/local/nginx
mkdir cert
cd cert/
mkdir ssl
```

推送本地证书文件至服务器新创建的目录：

推送方法比较多，可以选择使用FTP直接上传，不过需要先在服务器上配置好FTP。楼主比较喜欢简单的 SCP 命令，就是secure copy，是用来进行远程文件拷贝的。数据传输使用 ssh，并且和ssh 使用相同的认证方式，提供相同的安全保证。 

命令格式：
>scp \[参数\] <源地址（用户名@IP地址或主机名）>:<文件路径> <目的地址（用户名 @IP 地址或主机名）>:<文件路径> 

找到之前本地存放证书的目录 `/Users/Toy/ssl`，然后在终端执行脚本。

```javascriptt
scp /Users/Toy/ssl/cert.pem root@123.123.123.123:/usr/local/nginx/cert/ssl/cert.pem
scp /Users/Toy/ssl/cert.key root@123.123.123.123:/usr/local/nginx/cert/ssl/cert.key
```

以上两条命令即可搞定。

#### 2. 开放 443、80 端口
开放端口方法也很多，如果图省事，可以直接在阿里云控制台的安全组里面开放443端口或者80端口。（此方法开启后记得重启服务器，楼主开始也是通过此方法开启，然后一直不生效，坑了我2个小时😓）

当然也可以通过命令行的方式。
```javascriptt
// 启动防火墙
systemctl start firewalld.service
// 开启端口
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --zone=public --add-port=443/tcp --permanent
// 重启防火墙
firewall-cmd --reload
```

#### 3. 编辑 nginx.conf 文件，配置 SSL 规则
编辑 nginx.conf
```javascript
vi /usr/local/nginx/conf/nginx.conf
```

配置转发规则
```javascript
server {
    # 同时监听443和80端口
    listen       80;
    listen       443 ssl;
    server_name  api.tianxiaohu.online;
    # 证书目录
    ssl_certificate      /usr/local/nginx/cert/ssl/cert.pem;
    ssl_certificate_key  /usr/local/nginx/cert/ssl/cert.key;
    ssl_session_cache    shared:SSL:1m;
    ssl_session_timeout  5m;
    ssl_ciphers  HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers  on;
    # 转发规则
    location / {
        # 服务器接口地址
        proxy_pass http://127.0.0.1:8080; 
        add_header Access-Control-Allow-Origin *;
    }
}
```
重启 Nginx
```javascript
nginx -s reload
```

如此，如果操作不出意外，那么你就可以开心的使用 https 访问接口了。要不要点个右下角赏一杯咖啡？😃


