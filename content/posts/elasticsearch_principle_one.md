---
title: "扒一下ElasticSearch原理（一）| 索引管理"
date: 2022-10-02T10:21:24+08:00
draft: false
tags:
  [
    "elasticsearch",
    "distribte storage",
    "storage"
  ]
categories: ["distribute storge"]
---

## 索引管理 :pushpin:

### 索引介绍

### 创建索引
  
创建名字为`product`的索引，并指定id为1的文档的内容

``` shell
PUT /product/_doc/1
{
  "name": "computer"
}
```

![create_index](/images/elasticsearch_principle_one_create_index.png)  

### 修改索引

### 打开/关闭索引

### 删除索引

### 查看索引

### 批量索引文档

在源码中找到测试数据并下载

```
curl https://github.com/elastic/elasticsearch/blob/v6.8.19/docs/src/test/resources/accounts.json
```
将下载到的accounts.json文件批量导入到EleasticSearch中
```
curl -H "Content-Type: application/json" -XPOST "username:password@localhost:9200/bank/_bulk?pretty&refresh" --data-binary "@./accounts.json"

```

