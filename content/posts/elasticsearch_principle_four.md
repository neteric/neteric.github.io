---
title: "捋一捋ElasticSearch（四）| Lucene存储引擎"
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

## 1. 背景
Elasticsearch是一个基于Apache Lucene的开源搜索引擎，其依靠Lucene完成索引创建和搜索功能，可以将ElstaicSearch理解为是一个Lucene的分布式封装的搜索引擎。之所以ElasticSearch选用Lucene,是因为无论在开源还是专有领域，Lucene可以被认为是迄今为止最先进、性能最好的、功能最全的全文搜索引擎库。Lucene非常复杂，你需要深入了解检索的相关知识才能完全理解它是如何工作的，本文只作提纲挈领的总结一下Lucene的相关知识和流程

## 2. 倒排索引

### 2.1 什么是倒排索引

搜索的核心需求是全文检索，全文检索简单来说就是要在大量文档中找到包含某个单词出现的位置，在传统关系型数据库中，数据检索只能通过 like 来实现，例如需要在手机数据中查询名称包含苹果的手机，需要通过如下 sql 实现：

```shell
select * from phone_table where phone_brand like '%苹果%';
```
这种实现方式实际会存在很多问题：
- 无法使用数据库索引，需要全表扫描，性能差
- 搜索效果差，只能首尾位模糊匹配，无法实现复杂的搜索需求
- 无法得到文档与搜索条件的相关性
搜索的核心目标实际上是保证搜索的效果和性能，为了高效的实现全文检索，我们可以通过倒排索引来解决

倒排索引是区别于正排索引的概念
- 正排索引：是以文档对象的唯一ID 作为索引，以文档内容作为记录的结构
- 倒排索引：Inverted index，指的是将文档内容中的单词作为索引，将包含该词的文档 ID 作为记录的结构

![picture 6](/images/elasticsearch_principle_four_pic_zhengpai_index_vs_inverted_index.png)  


有了倒排索引，能快速、灵活地实现各类搜索需求。整个搜索过程中我们不需要做任何文本的模糊匹配。


### 2.2 倒排索引的结构

根据倒排索引的概念，最简单的我们可以用一个Map来描述这个结构。这个 Map 的 Key 的即是分词后的单词，这里的单词称为 Term，这一系列的 Term 组成了倒排索引的第一个部分 —— Term Dictionary (索引表，可简称为 Dictionary)
倒排索引的另一部分为 Posting List（记录表），也对应上述 Map 结构的 Value 集合。Posting List 由所有的 Term 对应的数据（Postings） 组成，它不仅仅包含文档 id 信息，还可能包含以下信息：

- 文档 id（DocId, Document Id），包含单词的所有文档唯一id，用于去正排索引中查询原始数据
- 词频（TF，Term Frequency），记录 Term 在每篇文档中出现的次数，用于后续相关性算分
- 位置（Position），记录 Term 在每篇文档中的分词位置（多个），用于做词语搜索（Phrase Query）
- 偏移（Offset），记录 Term 在每篇文档的开始和结束位置，用于高亮显示等


![picture 7](/images/elasticsearch_principle_four_pic_inverted_index_structure.png)  

全文搜索引擎在海量数据的情况下是需要存储大量的文本，所以面临以下问题：
- Dictionary 是非常大的（比如我们搜索中的一个字段可能有上千万个Term）
- Postings 可能会占据大量的存储空间（一个Term多的有几百万个doc）

因此上面说的基于 Map 的实现方式几乎是不可行的。在海量数据背景下，倒排索引的实现直接关系到存储成本以及搜索性能，为此，Lucene 引入了多种巧妙的数据结构和算法

## 3. Lucene核心数据结构

### 3.1 FST

Terms Dictionary 是存储在.tim 文件上的，当文档数量越来越多的时，Dictionary 中的 Term 也会越来越多，那查询效率必然也会逐渐变低，此需要一个很好的数据结构为 Dictionary 建构一个索引，这就是 Terms Index(.tip文件)，Lucene 采用了 FST 这个数据结构来实现这个索引，FST即Finite State Transducer（有限状态转换器）它具备以下特点：
- 给定一个 Input 可以得到一个 output，用法类似于 HashMap
- 共享前缀、后缀节省空间。FST 的内存消耗要比 HashMap 少很多
- 词查找复杂度为 O(len(str))
- 构建后不可变更

FST 详细内容可以参考
- https://www.shenyanchao.cn/blog/2018/12/04/lucene-fst/
- https://blog.burntsushi.net/transducers/


### 3.2 Term Directory
Terms Dictionary 通过 .tim 文件存储，
Terms Dictionary（索引表）存储所有的 Term 数据，同时它也是 Term 与 Postings 的关系纽带，存储了每个 Term 和其对应的 Postings 文件位置指针

![picture 8](/images/elasticsearch_principle_four_pic_term_directory_to_posting_list.png)  


### 3.3 Posting List

PostingList 包含文档 id、词频、位置等多个信息，这些数据之间本身是相对独立的，因此 Lucene 将 Postings List 拆成三个文件存储：
- .doc文件：记录 Postings 的 docId 信息和 Term 的词频
- .pay文件：记录 Payload 信息和偏移量信息
- .pos文件：记录位置信息

这个做法有没有点列式存储的味道？其好处也很明显：一来可以提高读取效率，因为基本所有的查询都会用 .doc 文件获取文档 id，但一般的查询仅需要用到 .doc 文件就足够了，只有对于近似查询等位置相关的查询才需要用位置相关数据， 二来这样存储数据很方便对数据进行压缩


> (TODO) 三个文件整体实现差不太多，这里以.doc 文件为例分析其实现。
.doc 文件存储的是每个 Term 对应的文档 Id 和词频。每个 Term 都包含一对 TermFreqs 和 SkipData 结构
其中 TermFreqs 存放 docId 和词频信息，SkipData 为跳表信息，用于实现 TermFreqs 内部的快速跳转

Posting List 采用多个文件进行存储，最终我们可以得到每个 Term 的如下信息：
- SkipOffset：用来描述当前 term 信息在 .doc 文件中跳表信息的起始位置。
- DocStartFP：是当前 term 信息在 .doc 文件中的文档 ID 与词频信息的起始位置。
- PosStartFP：是当前 term 信息在 .pos 文件中的起始位置。
- PayStartFP：是当前 term 信息在 .pay 文件中的起始位置。


## 4. Lucene的写入

## 5. Lucene的查询过程

```java
public TopDocs search(Query query, int n);
public Document doc(int docID);
public int count(Query query);
...
```
在介绍了索引表（Term Directory）和记录表（Posting List）的结构后，就可以得到 Lucene 倒排索引的查询步骤：
1. 通过 Term Index 数据（.tip文件）中的 StartFP 获取指定字段的 FST
2. 通过 FST 找到指定 Term 在 Term Dictionary（.tim 文件）可能存在的 Block
3. 将对应 Block 加载内存，遍历 Block 中的 Entry，通过后缀（Suffix）判断是否存在指定 Term
4. 存在则通过 Entry 的 TermStat 数据中各个文件的 FP 获取 Posting 数据
5. 如果需要获取 Term 对应的所有 DocId 则直接遍历 TermFreqs，如果获取指定 DocId 数据则通过 SkipData快速跳转

![picture 9](/images/elasticsearch_principle_four_pic_lucene_search_process.png)  

## 4. 动态索引更新（索引的写入）
### 4.1 Lucene的写入接口
Lucene中写操作主要是通过IndexWriter类实现，IndexWriter提供如下一些接口
```java
public long addDocument();
public long updateDocuments();
public long deleteDocuments();
public final void flush();
public final long prepareCommit();
public final long commit();
public final long rollback();
public void forceMerge();
public final void maybeMerge();
```

- addDocument：比较纯粹的一个API，就是向Lucene内新增一个文档。Lucene内部没有主键索引，所有新增文档都会被认为一个新的文档，分配一个独立的docId。 
- updateDocuments：更新文档，但是和数据库的更新不太一样。数据库的更新是查询后更新，Lucene的更新是查询后删除再新增。流程是先delete by term，后add document。但是这个流程又和直接先调用delete后调用add效果不一样，只有update能够保证在Thread内部删除和新增保证原子性，详细流程在下一章节会细说。 
- deleteDocuments：删除文档，支持两种类型删除，by term和by query。在IndexWriter内部这两种删除的流程不太一样，在下一章节再细说。 
- flush：触发强制flush，将所有Thread的In-memory buffer flush成segment文件，这个动作可以清理内存，强制对数据做持久化。 
- prepareCommit/commit/rollback：commit后数据才可被搜索，commit是一个二阶段操作，prepareCommit是二阶段操作的第一个阶段，也可以通过调用commit一步完成，rollback提供了回滚到last commit的操作。 
- maybeMerge/forceMerge：maybeMerge触发一次MergePolicy的判定，而forceMerge则触发一次强制merge。

通过这些接口可以完成单个文档的写入，更新和删除功能，包括了分词，倒排创建，正排创建等等所有搜索相关的流程。只要Doc通过IndexWriter写入后，后面就可以通过IndexSearcher搜索了,但有一些问题Lucene没有解决
- 上述操作是单机的，而不是我们需要的分布式
- 文档写入Lucene后并不是立即可查询的，需要生成完整的Segment后才可被搜索，如何在实时性和可靠性之间取得平衡
- Lucene不支持部分文档更新，但是这又是一个强需求，如何支持部分更新

### 4.2 ElasticSearch的写入

## 5. 搜索 （索引的读取）


## 参考文档
> - [1] [Lucene 源码分析](https://www.amazingkoala.com.cn/Lucene)
> - [2] [Lucene的词典FST深入剖析](https://www.shenyanchao.cn/blog/2018/12/04/lucene-fst/)
> - [3] [FST & tire-tree 应用场景](https://whatua.com/2019/10/26/fst-tire-tree-%E5%8F%8A%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF/)
> - [4] [KDTree wiki](https://en.wikipedia.org/wiki/K-d_tree)
> - [5] [Lucene Search 深入分析](https://zhuanlan.zhihu.com/p/509892041)
> - [6] [Lucene 解析 - IndexWriter](https://juejin.cn/post/6844903651878633479#heading-11)
