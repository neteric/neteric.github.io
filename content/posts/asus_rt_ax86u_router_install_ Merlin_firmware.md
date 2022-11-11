---
title: "华硕RT-AX86U路由器刷梅林固件"
date: 2022-10-19T10:21:24+08:00
draft: true
tags:
  [
    "router",
    "firmware",
    "asus"
  ]
categories: ["Router DIY"]
---


## 1. 官方固件

![图 1](/images/asus_rt_ax86u_router_update_kool_firmware_pic_officia_firmware.png)  

## 2. 刷机术语

双清就是要清除：1. nvram配置，2：JFFS分区文件。固件的很多设置都是储存在nvram中，例如拨号方式、拨号上网帐号密码、无线网络设置等；固件的很多文件是储存在JFFS分区的，例如流量分析储存的流量数据，SSL证书，UU加速器程序等。一般同类型固件互刷不需要进行双清，不同类型固件互刷视情况要进行双清，以保证路由器刷机后处于最佳工作状态。

如何双清路由器：进入【系统管理 】–【 恢复/导出/上传设置】，勾选恢复按钮旁的选择框，然后点击恢复按钮。

## 3. 刷机步骤
