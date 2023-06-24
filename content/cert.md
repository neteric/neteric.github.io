---
title: "免费SSL证书获取"
date: 2020-02-29T17:22:54+08:00
draft: false
tags: ["network","LoadBalancer", "LVS", "Nginx"]
categories: ["network"]
---

## ACME 原理

获取免费证书，let's encrypt 这个机构会验证你对域名的所有权，可以支持两种验证方式：

1. certbot在你网站的根目录（webroot）下创建一个隐藏文件，然后通过你的域名访问刚添加隐藏文件
2. certbot调用dns provider api，在你的域名下添加一个text类型dns记录，然后验证这个记录的值是否和添加的一致

## webroot 操作步骤

### 1. 安装客户端

``` bash
curl https://get.acme.sh | sh -s email=my@example.com
```

### 创建alias

创建 一个 shell 的 alias, 例如 .bashrc，方便你的使用:

```
 alias acme.sh=~/.acme.sh/acme.sh
```

### 2. 获取证书

你要为哪个域名添加ssl证书，就添加该域名的解析和ngxin配置，确保域名可以访问，配置好过后执行如下命令

``` bash
acme.sh --issue -d ${domain_name} --webroot   ${location_root_path}
```

### 3. 安装证书

``` bash
acme.sh --install-cert -d neteric.top --key-file  /work/neteric.top_nginx/neteric.top.key --fullchain-file  /work/neteric.top_nginx/neteric.top.cer  --reloadcmd     "nginx -s reload"
```

### 4. 配置自动更新

``` bash
crontabe -e
# 每周日凌晨3点检查更新
0 3  * * 7 /root/.acme.sh/acme.sh --cron --home /root/.acme.sh
```

## dns 操作步骤

### 1. 安装客户端

``` bash
yum install certbot
```

## 参考文章

1. <https://eff-certbot.readthedocs.io/en/stable/using.html#log-rotation>
