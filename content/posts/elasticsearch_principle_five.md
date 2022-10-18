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

## 1.ElasticSearch 文档索引过程（写）

### 1.1 数据持久化过程

![picture 7](/images/elasticsearch_principle_five_pic_es_data_dur_process.png)  

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

- **write**

一个新文档过来，会存储在 in-memory buffer 内存缓存区中，顺便会记录 Translog（Elasticsearch 增加了一个 translog ，或者叫事务日志，在每一次对 Elasticsearch 进行操作时均进行了日志记录）。 这时候数据还没到 segment ，是搜不到这个新文档的。数据只有被 refresh 后，才可以被搜索到
  ![picture 1](/images/elasticsearch_principle_five_pic_es_write_process1.png)  

- **refresh**

refresh 默认 1 秒钟，执行一次下图流程。ES 是支持修改这个值的，通过 index.refresh_interval 设置 refresh （冲刷）间隔时间。refresh 流程大致如下： in-memory buffer 中的文档写入到新的 segment 中，但 segment 是存储在文件系统的缓存中。此时文档可以被搜索到 最后清空 in-memory buffer。注意: Translog 没有被清空，为了将 segment 数据写到磁盘 文档经过 refresh 后， segment 暂时写到文件系统缓存，这样避免了性能 IO 操作，又可以使文档搜索到。refresh 默认 1 秒执行一次，性能损耗太大。一般建议稍微延长这个 refresh 时间间隔，比如 5 s。因此，ES 其实就是准实时，达不到真正的实时。
  ![picture 2](/images/elasticsearch_principle_five_pic_es_refresh_process1.png)  

- **flush**

每隔一段时间或者translog大到一定的阈值（默认512M）索引被刷新（flush）；一个新的 translog 被创建，并且一个全量提交被执行,
上个过程中 segment 在文件系统缓存中，会有意外故障文档丢失的风险，为了保证文档不会丢失，需要将文档写入磁盘。那么文档从文件缓存写入磁盘的过程就是 flush。写入磁盘后，清空 translog。具体内容如下：

- 所有在内存缓冲区的文档都被写入一个新的段，buffer pool缓冲区被清空
- 一个Commit Point被写入硬盘
- 文件系统缓存通过 fsync 被刷新（flush）
- 老的 translog 被删除。
  
![picture 3](/images/elasticsearch_principle_five_pic_es_flush_process1.png)  

- **megre**

由于自动刷新流程每秒会创建一个新的段 ，这样会导致短时间内的段数量暴增。而段数目太多会带来较大的麻烦。 每一个段都会消耗文件句柄、内存和cpu运行周期。更重要的是，每个搜索请求都必须轮流检查每个段；所以段越多，搜索也就越慢。 Elasticsearch通过在后台进行Merge Segment来解决这个问题。小的段被合并到大的段，然后这些大的段再被合并到更大的段, 这个过程是不是和LSM的SST Merge很相似？ 这可能就是大佬嘴里的那句话，技术都是相通的吧
当索引的时候，刷新（refresh）操作会创建新的段并将段打开以供搜索使用。合并进程选择一小部分大小相似的段，并且在后台将它们合并到更大的段中，这并不会中断索引和搜索

![picture 4](/images/elasticsearch_principle_five_pic_es_segment_merge_process.png)  

一旦合并结束：

- 新的段被刷新（flush）到了磁盘
- 写入一个包含新段且排除旧的和较小的段的新提交点
- 新的段被打开用来搜
- 老的段被删除

![picture 5](/images/elasticsearch_principle_five_pic_es_index_alltotal_process.png)  

### 1.3 分布式写入总结

#### coordination node

1. Ingest Pipeline

    在这一步可以对原始文档做一些处理，比如HTML解析，自定义的处理，具体处理逻辑可以通过插件来实现。在Elasticsearch中，由于Ingest Pipeline会比较耗费CPU等资源，可以设置专门的Ingest Node，专门用来处理Ingest Pipeline逻辑。 如果当前Node不能执行Ingest Pipeline，则会将请求发给另一台可以执行Ingest Pipeline的Node
2. Auto Create Index

    判断当前Index是否存在，如果不存在，则需要自动创建Index，这里需要和Master交互。也可以通过配置关闭自动创建Index的功能
3. Set Routing

    设置路由条件，如果Request中指定了路由条件，则直接使用Request中的Routing，否则使用Mapping中配置的，如果Mapping中无配置，则使用默认的_id字段值。 在这一步中，如果没有指定id字段，则会自动生成一个唯一的_id字段，目前使用的是UUID
4. Construct BulkShardRequest
  
    由于Bulk Request中会包括多个(Index/Update/Delete)请求，这些请求根据routing可能会落在多个Shard上执行，这一步会按Shard挑拣Single Write Request，同一个Shard中的请求聚集在一起，构建BulkShardRequest，每个BulkShardRequest对应一个Shard
5. Send Request To Primary

   这一步会将每一个BulkShardRequest请求发送给相应Shard的Primary Node

#### primary node

1. Index or Update or Delete

   循环执行每个Single Write Request，对于每个Request，根据操作类型(CREATE/INDEX/UPDATE/DELETE)选择不同的处理逻辑。 其中，Create/Index是直接新增Doc，Delete是直接根据_id删除Doc，Update会稍微复杂些，我们下面就以Update为例来介绍
2. Translate Update To Index or Delete

   这一步是Update操作的特有步骤，在这里，会将Update请求转换为Index或者Delete请求。首先，会通过GetRequest查询到已经存在的同_id Doc（如果有）的完整字段和值（依赖_source字段），然后和请求中的Doc合并。同时，这里会获取到读到的Doc版本号，记做V1

3. Parse Doc
  
    这里会解析Doc中各个字段。生成ParsedDocument对象，同时会生成uid Term。在Elasticsearch中，_uid = type #_id，对用户，_Id可见，而Elasticsearch中存储的是_uid。这一部分生成的ParsedDocument中也有Elasticsearch的系统字段，大部分会根据当前内容填充，部分未知的会在后面继续填充ParsedDocument
4. Update Mapping

    Elasticsearch中有个自动更新Mapping的功能，就在这一步生效。会先挑选出Mapping中未包含的新Field，然后判断是否运行自动更新Mapping，如果允许，则更新Mapping
5. Get Sequence Id and Version
  
    由于当前是Primary Shard，则会从SequenceNumber Service获取一个sequenceID和Version。SequenceID在Shard级别每次递增1，SequenceID在写入Doc成功后，会用来初始化LocalCheckpoint。Version则是根据当前Doc的最大Version递增
6. Add Doc To Lucene

   这一步开始的时候会给特定_uid加锁，然后判断该_uid对应的Version是否等于之前Translate Update To Index步骤里获取到的Version，如果不相等，则说明刚才读取Doc后，该Doc发生了变化，出现了版本冲突，这时候会抛出一个VersionConflict的异常，该异常会在Primary Node最开始处捕获，重新从“Translate Update To Index or Delete”开始执行。 如果Version相等，则继续执行，如果已经存在同id的Doc，则会调用Lucene的UpdateDocument(uid, doc)接口，先根据uid删除Doc，然后再Index新Doc。如果是首次写入，则直接调用Lucene的AddDocument接口完成Doc的Index，AddDocument也是通过UpdateDocument实现。 这一步中有个问题是，如何保证Delete-Then-Add的原子性，怎么避免中间状态时被Refresh？答案是在开始Delete之前，会加一个Refresh Lock，禁止被Refresh，只有等Add完后释放了Refresh Lock后才能被Refresh，这样就保证了Delete-Then-Add的原子性。 Lucene的UpdateDocument接口中就只是处理多个Field，会遍历每个Field逐个处理，处理顺序是invert index，store field，doc values，point dimension，具体参考前文
7. Write Translog
  
    写完Lucene的Segment后，会以keyvalue的形式写TransLog，Key是_id，Value是Doc内容。当查询的时候，如果请求是GetDocByID，则可以直接根据_id从TransLog中读取到，满足NoSQL场景下的实时性要去。 需要注意的是，这里只是写入到内存的TransLog，是否Sync到磁盘的逻辑还在后面。 这一步的最后，会标记当前SequenceID已经成功执行，接着会更新当前Shard的LocalCheckPoint
8. Renew Bulk Request

    这里会重新构造Bulk Request，原因是前面已经将UpdateRequest翻译成了Index或Delete请求，则后续所有Replica中只需要执行Index或Delete请求就可以了，不需要再执行Update逻辑，一是保证Replica中逻辑更简单，性能更好，二是保证同一个请求在Primary和Replica中的执行结果一样
9. Flush Translog

    这里会根据TransLog的策略，选择不同的执行方式，要么是立即Flush到磁盘，要么是等到以后再Flush。Flush的频率越高，可靠性越高，对写入性能影响越大
10. Send Requests To Replicas

    这里会将刚才构造的新的Bulk Request并行发送给多个Replica，然后等待Replica的返回，这里需要等待所有Replica返回后（可能有成功，也有可能失败），Primary Node才会返回用户。如果某个Replica失败了，则Primary会给Master发送一个Remove Shard请求，要求Master将该Replica Shard从可用节点中移除。 这里，同时会将SequenceID，PrimaryTerm，GlobalCheckPoint等传递给Replica。 发送给Replica的请求中，Action Name等于原始ActionName + [R]，这里的R表示Replica。通过这个[R]的不同，可以找到处理Replica请求的Handler
11. Receive Response From Replicas

    Replica中请求都处理完后，会更新Primary Node的LocalCheckPoint

#### Replica Node

1. Index or Delete 根据请求类型是Index还是Delete，选择不同的执行逻辑。这里没有Update，是因为在Primary Node中已经将Update转换成了Index或Delete请求了
2. Parse Doc
3. Update Mapping 以上都和Primary Node中逻辑一致
4. Get Sequence Id and Version Primary Node中会生成Sequence ID和Version，然后放入ReplicaRequest中，这里只需要从Request中获取到就行
5. Add Doc To Lucene 由于已经在Primary Node中将部分Update请求转换成了Index或Delete请求，这里只需要处理Index和Delete两种请求，不再需要处理Update请求了。比Primary Node会更简单一些
6. Write Translog
7. Flush Translog 以上都和Primary Node中逻辑一致。

## 2. ElasticSearch 文档搜索过程（读）

### 2.1 搜索过程拆解

几乎所有的搜索系统一般都是两阶段查询，第一阶段通过倒排索引等查询到匹配的DocID，第二阶段再查询DocID对应的完整文档，这种在Elasticsearch中称为query_then_fetch

![图 1](/images/elasticsearch_principle_five_pic_pic_es_read_simple_process.png)  

1. 在初始**查询阶段**时，查询会广播到索引中每一个分片拷贝（主分片或者副本分片）。 每个分片在本地执行搜索并构建一个匹配文档的大小为 from + size 的优先队列
2. 每个分片返回各自优先队列中 所有文档的 ID 和排序值给协调节点，它合并这些值到自己的优先队列中来产生一个全局排序后的结果列表。
3. 接下来就是**取回阶段**，协调节点辨别出哪些文档需要被取回并向相关的分片提交多个 GET 请求。每个分片加载并丰富文档，如果有需要的话，接着返回文档给协调节点。一旦所有的文档都被取回了，协调节点返回结果给客户端

### 2.2 分布式搜索总结

![picture 6](/images/elasticsearch_principle_five_pic_es_totalread_process.png)  

#### search coordination node

1. Get Remote Cluster Shard

   判断是否需要跨集群访问，如果需要，则获取到要访问的Shard列表
2. Get Search Shard Iterator

   获取当前Cluster中要访问的Shard，和上一步中的Remote Cluster Shard合并，构建出最终要访问的完整Shard列表。 这一步中，会根据Request请求中的参数从Primary Node和多个Replica Node中选择出一个要访问的Shard
3. For Every Shard:Perform

   遍历每个Shard，对每个Shard执行后面逻辑
4. Send Request To Query Shard

   将查询阶段请求发送给相应的Shard
5. Merge Docs

   上一步将请求发送给多个Shard后，这一步就是异步等待返回结果，然后对结果合并。这里的合并策略是维护一个Top N大小的优先级队列，每当收到一个shard的返回，就把结果放入优先级队列做一次排序，直到所有的Shard都返回。 翻页逻辑也是在这里，如果需要取Top 30~ Top 40的结果，这个的意思是所有Shard查询结果中的第30到40的结果，那么在每个Shard中无法确定最终的结果，每个Shard需要返回Top 40的结果给Client Node，然后Client Node中在merge docs的时候，计算出Top 40的结果，最后再去除掉Top 30，剩余的10个结果就是需要的Top 30~ Top 40的结果。 上述翻页逻辑有一个明显的缺点就是每次Shard返回的数据中包括了已经翻过的历史结果，如果翻页很深，则在这里需要排序的Docs会很多，比如Shard有1000，取第9990到10000的结果，那么这次查询，Shard总共需要返回1000 * 10000，也就是一千万Doc，这种情况很容易导致OOM。 另一种翻页方式是使用search_after，这种方式会更轻量级，如果每次只需要返回10条结构，则每个Shard只需要返回search_after之后的10个结果即可，返回的总数据量只是和Shard个数以及本次需要的个数有关，和历史已读取的个数无关。这种方式更安全一些，推荐使用这种。 如果有aggregate，也会在这里做聚合，但是不同的aggregate类型的merge策略不一样
6. Send Request To Fetch Shard

   选出Top N个Doc ID后发送给这些Doc ID所在的Shard执行Fetch Phase，最后会返回Top N的Doc的内容。

#### Query Phase

> 第一阶段查询的步骤

1. Create Search Context

   创建Search Context，之后Search过程中的所有中间状态都会存在Context中，这些状态总共有50多个，具体可以查看DefaultSearchContext或者其他SearchContext的子类
2. Parse Query

   解析Query的Source，将结果存入Search Context。这里会根据请求中Query类型的不同创建不同的Query对象，比如TermQuery、FuzzyQuery等，最终真正执行TermQuery、FuzzyQuery等语义的地方是在Lucene中。 这里包括了dfsPhase、queryPhase和fetchPhase三个阶段的preProcess部分，只有queryPhase的preProcess中有执行逻辑，其他两个都是空逻辑，执行完preProcess后，所有需要的参数都会设置完成。 由于Elasticsearch中有些请求之间是相互关联的，并非独立的，比如scroll请求，所以这里同时会设置Context的生命周期。 同时会设置lowLevelCancellation是否打开，这个参数是集群级别配置，同时也能动态开关，打开后会在后面执行时做更多的检测，检测是否需要停止后续逻辑直接返回
3. Get From Cache

   判断请求是否允许被Cache，如果允许，则检查Cache中是否已经有结果，如果有则直接读取Cache，如果没有则继续执行后续步骤，执行完后，再将结果加入Cache
4. Add Collectors Collector

   主要目标是收集查询结果，实现排序，对自定义结果集过滤和收集等。这一步会增加多个Collectors，多个Collector组成一个List。 FilteredCollector：先判断请求中是否有Post Filter，Post Filter用于Search，Agg等结束后再次对结果做Filter，希望Filter不影响Agg结果。如果有Post Filter则创建一个FilteredCollector，加入Collector List中。 PluginInMultiCollector：判断请求中是否制定了自定义的一些Collector，如果有，则创建后加入Collector List。 MinimumScoreCollector：判断请求中是否制定了最小分数阈值，如果指定了，则创建MinimumScoreCollector加入Collector List中，在后续收集结果时，会过滤掉得分小于最小分数的Doc。 EarlyTerminatingCollector：判断请求中是否提前结束Doc的Seek，如果是则创建EarlyTerminatingCollector，加入Collector List中。在后续Seek和收集Doc的过程中，当Seek的Doc数达到Early Terminating后会停止Seek后续倒排链。 CancellableCollector：判断当前操作是否可以被中断结束，比如是否已经超时等，如果是会抛出一个TaskCancelledException异常。该功能一般用来提前结束较长的查询请求，可以用来保护系统。 EarlyTerminatingSortingCollector：如果Index是排序的，那么可以提前结束对倒排链的Seek，相当于在一个排序递减链表上返回最大的N个值，只需要直接返回前N个值就可以了。这个Collector会加到Collector List的头部。EarlyTerminatingSorting和EarlyTerminating的区别是，EarlyTerminatingSorting是一种对结果无损伤的优化，而EarlyTerminating是有损的，人为掐断执行的优化。 TopDocsCollector：这个是最核心的Top N结果选择器，会加入到Collector List的头部。TopScoreDocCollector和TopFieldCollector都是TopDocsCollector的子类，TopScoreDocCollector会按照固定的方式算分，排序会按照分数+doc id的方式排列，如果多个doc的分数一样，先选择doc id小的文档。而TopFieldCollector则是根据用户指定的Field的值排序
5. Lucene Search

   这一步会调用Lucene中IndexSearch的search接口，执行真正的搜索逻辑。每个Shard中会有多个Segment，每个Segment对应一个LeafReaderContext，这里会遍历每个Segment，到每个Segment中去Search结果，然后计算分数。 搜索里面一般有两阶段算分，第一阶段是在这里算的，会对每个Seek到的Doc都计算分数，为了减少CPU消耗，一般是算一个基本分数。这一阶段完成后，会有个排序。然后在第二阶段，再对Top 的结果做一次二阶段算分，在二阶段算分的时候会考虑更多的因子。二阶段算分在后续操作中。 具体请求，比如TermQuery、WildcardQuery的查询逻辑都在Lucene中
6. Rescore

   根据Request中是否包含rescore配置决定是否进行二阶段排序，如果有则执行二阶段算分逻辑，会考虑更多的算分因子。二阶段算分也是一种计算机中常见的多层设计，是一种资源消耗和效率的折中。 Elasticsearch中支持配置多个Rescore，这些rescore逻辑会顺序遍历执行。每个rescore内部会先按照请求参数window选择出Top window的doc，然后对这些doc排序，排完后再合并回原有的Top 结果顺序中
7. Suggest Execute

   如果有推荐请求，则在这里执行推荐请求。如果请求中只包含了推荐的部分，则很多地方可以优化。推荐不是今天的重点，这里就不介绍了，后面有机会再介绍
8. aggregation::execute()

   如果含有聚合统计请求，则在这里执行。Elasticsearch中的aggregate的处理逻辑也类似于Search，通过多个Collector来实现。在Client Node中也需要对aggregation做合并。aggregate逻辑更复杂一些，就不在这里赘述了，后面有需要就再单独开文章介绍。 上述逻辑都执行完成后，如果当前查询请求只需要查询一个Shard，那么会直接在当前Node执行Fetch Phase

#### Fetch Phase

> 第二阶段Fetch的步骤

Elasticsearch作为搜索系统时除了Query阶段外，还会有一个Fetch阶段，这个Fetch阶段在数据库类系统中是没有的。搜索系统中额外增加Fetch阶段的原因是搜索系统中数据分布导致的，在搜索中，数据通过routing分Shard的时候，只能根据一个主字段值来决定，但是查询的时候可能会根据其他非主字段查询，那么这个时候所有Shard中都可能会存在相同非主字段值的Doc，所以需要查询所有Shard才能不会出现结果遗漏。同时如果查询主字段，那么这个时候就能直接定位到Shard，就只需要查询特定Shard即可，这个时候就类似于数据库系统了。另外，数据库中的二级索引又是另外一种情况，但类似于查主字段的情况，这里就不多说了

基于上述原因，第一阶段查询的时候并不知道最终结果会在哪个Shard上，所以每个Shard中都需要查询完整结果，比如需要Top 10，那么每个Shard都需要查询当前Shard的所有数据，找出当前Shard的Top 10，然后返回给Client Node。如果有100个Shard，那么就需要返回100 * 10 = 1000个结果，而Fetch Doc内容的操作比较耗费IO和CPU，如果在第一阶段就Fetch Doc，那么这个资源开销就会非常大。所以，一般是当Client Node选择出最终Top N的结果后，再对最终的Top N读取Doc内容。通过增加一点网络开销而避免大量IO和CPU操作，这个折中是非常划算的

Fetch阶段的目的是通过DocID获取到用户需要的完整Doc内容。这些内容包括了DocValues，Store，Source，Script和Highlight等

## 参考文档

> [1] [Elasticsearch内核解析](https://zhuanlan.zhihu.com/p/34674517)
