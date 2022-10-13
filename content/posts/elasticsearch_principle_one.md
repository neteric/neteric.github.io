---
title: "扒一下ElasticSearch原理（一）"
date: 2022-10-02T10:21:24+08:00
draft: false
tags:
  [
    "elasticsearch",
    "distribte storage",
    "storage"
  ]
categories: ["distribute"]
---

### :pushpin: 索引管理
- 创建所索引
  
创建名字为`product`的索引，并指定id为1的文档的内容

``` shell
PUT /product/_doc/1
{
  "name": "computer"
}
```

![create_index](/images/elasticsearch_principle_one_create_index.png)  

-  **查看索引**


- **批量索引文档**

在源码中找到测试数据并下载

```
curl https://github.com/elastic/elasticsearch/blob/v6.8.19/docs/src/test/resources/accounts.json
```
将下载到的accounts.json文件批量导入到EleasticSearch中
```
curl -H "Content-Type: application/json" -XPOST "username:password@localhost:9200/bank/_bulk?pretty&refresh" --data-binary "@./accounts.json"

```

### :pushpin: 数据查询
- **查询所有**
  
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


- **分页查询**

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

- **指定字段查询**

```
GET /bank/_search
{
  "query": { "match": { "address": "mill lane" } }
}

```

- **多条件查询**

![picture 24](/images/elasticsearch_principle_one_search_by_field.png)  


- **聚合查询（Aggregation）**
  
  在SQL中有group by，在ES中它叫Aggregation，即聚合运算
