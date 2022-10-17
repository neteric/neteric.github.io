---
title: "捋一捋ElasticSearch（五）| 分布式搜索原理总结"
date: 2022-10-05T10:21:24+08:00
draft: false
tags:
  [
    "elasticsearch",
    "distribte storage",
    "storage"
  ]
categories: ["distribute storge"]
---

## 1. 索引文档流程详解（写）

### 1.1 elasticsearch文档索引的整体流程

![图 4](/images/elasticsearch_principle_five_pic_doc_index_total_process.png)  

1. 客户端发送文档索引请求到任意节点，这时候该节点被称为协调节点，协调节点负责根据文档->分片的路由规则计算出主分片，然后从集群的Meta中找到该分片的节点信息
2. 协调节点将请求转发到主分片所在的node节点的Lucene实例上
3. 当分片所在的节点接收到来自协调节点的请求后，会调用Lucene的`addDocument`接口并传入文档数据来创建索引，此时文档数据和对应的索引都在Buffer Pool中
4. 为了降低对磁盘io的影响，在内存Buffer Pool中建立好了索引过后并不会立马把buffer pool中是索引flush到硬盘，而是会把buffer pool的内容**追加**到tranlog文件中，然后返回给协调节点写入成功的响应，此时该索引对客户端是unsearchable的
5. 为了让客户端能尽快的搜索到刚才提交成功的文档，而后默认每一秒中进行一次调用Lucene的`commit`接口的操作将Buffer Pool内的文档和索引commit到硬盘形成segment，一旦segment形成那么Lucene的IndexSearch模块就可以拿到文件句柄并据此成功的搜到该文档的内容。这个过程，在ElasticSearch中叫做**refresh**，一旦segment形成buffer pool的内存空间就会立马释放
6. refresh的完成后，虽然segment文件已经形成但此时segment的数据还在filesystem的pagecache中没有真正落盘，默认情况下，操作系统会按照自己的策略来flush pagecache中的数据到硬盘，当然elasticsearch的用户也可以手动调用[flush api](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/indices-flush.html)来刷盘，但一般不建议这样做，因为即使系统这时候挂掉因为有了tranlog的保障，数据也可以恢复的
7. 每隔一段时间或者tranlog大到一定的尺寸过后， Lucene会触发调用`flush`接口，将所有Thread的In-memory buffer flush成segment文件，然后老的translog被删除，page cache 被清空
8. 段合并

> 因为3的原因，ElasticSearch被称为是近实时的搜索引擎。但当Elasticsearch作为NoSQL数据库时，查询方式是GetById，这种查询可以直接从TransLog中查询，这时候就成了RT（Real Time）实时系统。

### 1.2 分步骤看数据持久化过程

> Lucene数据持久化过程：write -> refresh -> flush -> merge

- write
- refresh
- flush
- megre
  
### 1.3 Lucene本地写过过程

### 1.4 写操作的关键点

### 1.5 ElasticSearch 写入请求的类型

## 2. 搜索文档流程（读）
