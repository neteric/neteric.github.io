---
title: "捋一捋ElasticSearch（一）| 基本使用"
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

> 罗列一些DSL的基本使用，后续遇到相关问题可以完善和查询使用,本文的用法，可以参考[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/rest-apis.html)

## :pushpin: 1. 索引管理

### 1.1 索引介绍

### 1.2 创建索引
  
创建名字为`product`的索引，并指定id为1的文档的内容

``` shell
PUT /product/_doc/1
{
  "name": "computer"
}
```

![create_index](/images/elasticsearch_principle_one_create_index.png)  

### 1.3 修改索引

### 1.4 打开/关闭索引

### 1.5 删除索引

### 1.6 查看索引

### 1.7 批量索引文档

### 1.8 索引设置


在源码中找到测试数据并下载

```
curl https://github.com/elastic/elasticsearch/blob/v6.8.19/docs/src/test/resources/accounts.json
```

将下载到的accounts.json文件批量导入到EleasticSearch中

```
curl -H "Content-Type: application/json" -XPOST "username:password@localhost:9200/bank/_bulk?pretty&refresh" --data-binary "@./accounts.json"

```

## :scroll: 2. 文档管理

## :mag_right: 3.搜索

### 3.1 查询所有
  
查询index名为bank的数据，`match_all`表示查询所有的数据，`sort`即按照什么字段排序

```
GET /bank/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "desc" }
  ]
}
```

查询结果如下：

![picture 23](/images/elasticsearch_principle_one_bank_search_all.png)  

相关字段解释：

```
  - took – Elasticsearch运行查询所花费的时间（以毫秒为单位） 
  - timed_out –搜索请求是否超时 
  - _shards - 搜索了多少个分片，成功、失败或跳过了多少个分片
  - max_score – 找到的最相关文档的分数 
  - hits.total.value - 找到了多少个匹配的文档 
  - hits._score - 文档的相关性得分（使用match_all时不适用）
```

### 3.2 分页查询

从第`3`页记录开始查询，每页`5`条数据,与Sql语句中的offset和limit类似

```
GET /bank/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ],
  "from": 10,
  "size": 5
}
```

### 3.3 指定字段查询

```
GET /bank/_search
{
  "query": { "match": { "address": "mill lane" } }
}

```

### 3.4 多条件查询

![picture 24](/images/elasticsearch_principle_one_search_by_field.png)  

### 3.5 聚合查询（Aggregation）
  
  在SQL中有group by，在ES中它叫Aggregation，即聚合运算


## 参考文档

> [1] [官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/rest-apis.html)