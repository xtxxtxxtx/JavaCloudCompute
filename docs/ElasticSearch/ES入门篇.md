@[toc]
#### 1、ES是什么
- 一个分布式实时文档存储，每个字段可以被索引和搜索
- 一个分布式实时分析搜索引擎
- 能胜任上百个服务节点的扩展，并且支持PB级别的结构化或者是非结构化数据
- 基于Lucene，隐藏复杂性，提供简单易用的Restful API接口、Java API接口。ES是一个可高度扩展的全文搜索和分析引擎，可以快速地、近乎实时的存储、查询和分析大量数据
#### 2、ES基本结构
##### 2.1、结构图
![在这里插入图片描述](https://img-blog.csdnimg.cn/ae530ee49a184643a6afb161c44154f5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5ZSJLg==,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

##### 2.2、基本概念
- 集群（cluster）
ES集群由若干个节点组成，这些节点在同一个网络内，集群名字相同
- 节点（node）
一个ES实例，本质上是一个java进程，生产环境一般建议都是一台机器上运行一个ES实例。节点分类如下：

1. master 节点：集群中的一个节点会被选为 master 节点，负责管理集群范畴的变更，例如创建或删除索引，添加节点到集群或从集群删除节点。master 节点无需参与文档层面的变更和搜索，这意味着仅有一个 master 节点并不会因流量增长而成为瓶颈。任意一个节点都可以成为 master 节点
2. data 节点：持有数据和倒排索引。默认情况下，每个节点都可通过配置文件中的 node.data 属性为 true (默认)成为数据节点，如果需要一个专门的主节点，应将其 node.data 属性设置为 false
3. client 节点：如果将 node.master 属性和 node.data 属性全部设置为 false，那么该节点就是一个客户端节点，扮演一个负载均衡的角色，将到来的请求路由到集群中的各个节点

- 索引（index）

文档的容器，一类文档的集合

- 分片（shard）

单个节点由于物理机硬件限制，存储的文档是有限的，如果一个索引包含海量文档，则不能在单个节点存储。ES提供分片机制，同一个索引可以存储在不同分片中，这些分片又可以存储在集群中不同节点上。分片分为 primary shard 和 replica shard

1. 主分片与从分片关系：从分片只是主分片的一个副本，用于提供数据的冗余副本；
2. 从分片应用：硬件故障时提供数据保护，同时服务于搜索和检索这种只读请求；
3. 是否可变：索引中的主分片数量在索引创建后就固定下来，但是从分片的数量可以随时改变。

- 文档（document）

可搜索的最小单元，json格式保存

##### 2.3、和关系型数据库概念类比

| RDBMS  | ES       |
| ------ | -------- |
| Table  | Index    |
| Row    | Document |
| Column | Field    |
| Schema | Mapping  |
| SQL    | DSL      |

#### 3、ES原理

##### 3.1、Node节点管理

###### 3.1.1、多节点集群方案

多节点集群方案提高了整个系统的并发处理能力。

当索引一个文档的时候，文档会被存储到一个主分片中，那么如何判断一个文档存储到哪个主分片中呢？

> shard = hash(routing) % number_of_primary_shards

routing 是一个可变值，默认是文档的 _id，也可以设置成一个自定义的值。number_of_prmary_shards，所以在创建索引时候就确定以好主分片的数量。
![在这里插入图片描述](https://img-blog.csdnimg.cn/95bcf6b61aa5430a80e9e5153ca2d585.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5ZSJLg==,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)
###### 3.1.2、协调节点

节点分为主节点、数据节点和客户端节点(只做请求的分发和汇总)。每个节点都可以接受客户端的请求，每个节点都知道集群中任意文档位置，所以可以直接将请求转发到需要的节点上，当接受请求后，节点变为协调节点，参与转发。

以局部更新文档为例：

1、客户端向 Node1 发送更新请求，Node1 根据 _id 确认属于P1

2、将请求转发到主分片P1所在的 Node3

3、Node3 从主分片检索文档，修改 _source 字段中的 JSON，并且尝试重新索引主分片的文档，如果文档已经被另一个进程修改，会重试步骤3，超过 retry_on_conflict 次后放弃

4、如果 Node3 成功的更新文档，它将新版本的文档并行转发到 Node1 和 Node2 的副本分片去重新建立索引，一旦所有副本分片都返回成功，Node3 向协调节点返回成功，协调节点向客户端返回成功。

> 基于文档的复制
>
> 当主分片把更改转发到副本分片时，不会转发更新请求。相反转发完成文档的新版本。需要注意的是这些更改将会异步转发到副本分片，并且不能保证它们以发送它们相同的顺序到达。如果ES仅转发更改请求，则可能以错误的顺序应用更改，导致得到损坏的文档。

###### 3.1.3、节点故障转移

原集群：索引设置的是 3 主 1 从(索引设置的3个主分片，每个主分片一个从分片)。Node1 是 master 节点，P0、P1、P2 是主分片，R0、R1、R2 是从分片。
![在这里插入图片描述](https://img-blog.csdnimg.cn/6795067d773b4ef7ad7d3405e8efb346.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5ZSJLg==,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)


1. Node1 宕机，重选 master 节点，Node2 成为 master 节点；
2. Node3 上的 R0 上升成 P0，此时主分片恢复；
3. R0、R1 分片在 Node2，Node3 分片，集群完全恢复。

##### 3.2、shard分片原理

###### 3.2.1、文本可被搜索：分词器+倒排索引

问题1：文本如何分析？

为了提高可检索性(比如希望大小写单词全部返回)，应当先用分词器，分析文档再对其索引，分析包括两个部分：

- 将句子词条化为独立的单词
- 将单词规范化为标准形式

默认情况下，ES 使用标准分析器，使用了：

- 标准分词器以单词为界来切词
- 小写词条过滤器来转换单词

还有很多可用的分析器不在此列举，为了实现查询时能得到对应的结果，查询时应使用与索引时一致的分析器，对文档进行分析。

> 精确值一般不会被分词器分词，全文本需要分词器处理

问题2：倒排索引是什么？如何提升搜索速度？

- 下图就是一个倒排索引：

  | docid | Age  | Sex  |
  | ----- | ---- | ---- |
  | 1     | 18   | 女   |
  | 2     | 20   | 女   |
  | 3     | 18   | 男   |

  ID 是文档 id ，那么建立的索引如下：

  Age：

  | term | PostingList |
  | ---- | ----------- |
  | 18   | [1, 3]      |
  | 20   | [2]         |

  Sex:

  | term | PostingList |
  | ---- | ----------- |
  | 男   | [3]         |
  | 女   | [1, 2]      |

  1、PostingList 是一个 int 的数组，存储所有符合某个 term 的文档 id

  2、term 数量很多，查找某个指定的 term 会变慢，因此采用 Term Dictionary，对于 term 进行排序，采用二分查找查询 logN次才能查到

  3、即便变成查询 logN 次，但是由于数据不可能全量放在内存，因此还是存在磁盘操作，磁盘操作导致耗时增加，因此通过 trie 树，构建 Term Index。一般的 Trie树实现，例如 abc、ab，cd，c，mn 这些term，建立 trie 树，如下所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/3ad0739213da4714bcae273b4caaeda7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5ZSJLg==,size_17,color_FFFFFF,t_70,g_se,x_16#pic_center)


该 trie 树不会存储所有的 terms。当查找对应的 term，根据 trie 树找到 Term Dictionary 对应的 offset， 从偏移的位置往后顺序查找。除此以外，term index 在内存中是以 FST(finite state transducers)的形式保存的，其特点是非常节省内存的。Term Dictionary 在磁盘上是以分 block 的方式保存的，一个 block 内部利用公共前缀压缩，比如都是 Ab 开头的单词就可以把 Ab省去。这样 term Dictionary 可以比 b-tree 更加节约磁盘空间。

> 由此可以看出 MySQL 和 ES 的不同，MySQL 采用 B+ 树建立索引，相当于做了 Term Dictionary 这一层。但是 es 还支持了 term index，并且采用有效的算法，能够快速定位到对应的 term，减少随机查询磁盘的次数。

问题3：倒排索引的特点？

倒排索引被写入磁盘后是不可以改变的；它永远不会修改，不变性有重要的价值：

1、不需要锁：如果一直不更新索引，就不需要担心多进程同时修改数据的问题

2、一旦索引被读入内核的文件系统缓存，便会留在哪里，由于其不变性。只要文件系统缓存中还有足够的空间，那么大部分读请求会直接请求内存，而不会命中磁盘。这提供了很大的性能提升。

3、其它缓存(像filter缓存)，在索引的生命周期内始终有效，它们不需要再每次数据改变时被重建，因为数据不会变化

4、写入单个大的倒排索引允许数据被压缩，减少磁盘 IO 和需要被缓存到内存的索引的使用量

当然，一个不变的索引也有不好的地方，主要事实是它是不可变的，无法进行修改。如果需要让一个新的文档可被搜索，需要重建珍整个索引，要么对一个索引所能包含的数据量造成了很大的限制，要么对索引可被更新的频率造成了很大的限制。

###### 3.2.2、动态更新索引

上面讲到倒排索引的不变性，那么如果文档更新，该如何更新倒排索引呢？用更多的索引。

1. 利用新增的索引来反映修改
2. 查询时从旧的到新的依次查询
3. 最后来一个结果合并

ES 底层基于 Lucene，最核心的概念就是 Segment(段)，每个段本身就是一个倒排索引，ES 的 index 由多个段的集合和 commit point(提交点，一个列出了所有已知段的文件)文件组成。

> 一个 Lucene 索引，在 ES 称作分片，一个 ES 索引是分片的集合，当 ES 在索引中搜索的时候，发送查询到每一个属于索引的分片(Lucene 索引)，然后像分布式检索提到的集群，合并每个分片的结果到一个全局的结果集。
![在这里插入图片描述](https://img-blog.csdnimg.cn/0880efe9479e48f78d65f8a84b1e44bf.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5ZSJLg==,size_18,color_FFFFFF,t_70,g_se,x_16#pic_center)


新的文档首先被添加到内存索引缓存中，然后写入到一个基于磁盘的段，如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/d22a8f5c79dc488aa42d296142275dbf.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5ZSJLg==,size_16,color_FFFFFF,t_70,g_se,x_16#pic_center)
											一个 Lucene 索引(三个段加上一个提交点) + 内存索引缓存

![在这里插入图片描述](https://img-blog.csdnimg.cn/2af635ac723344798c50680dd767696d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5ZSJLg==,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)
​														一个 Lucene 索引(四个段加上一个提交点)

当一个查询触发时，按段对所有已知的段查询，词项统计会对所有段的结果进行聚合，以保证每个词和每个文档的关联都被准确计算。这种方式可以用相对较低的成本将新文档添加到索引。那聚合过程中，对于更新的或者删除的数据如何处理的呢？

段是不可改变的，所以既不能从把文档从旧的段中移除，也不能修改旧的段来进行反映文档的更新。取而代之的是每个提交点包含一个 .del 文件，文件中会列出这些被删除文档的段信息，当一个文档被 "删除" 时，它实际上只是在 .del 文件中被标记删除。一个被标记删除的文档仍然可以被查询匹配时，但它会在最终结果被返回前从结果集中移除，文档更新也是类似的操作方式；当一个文档被更新时，旧版本文档被标记删除，文档的新版本被索引到一个新的段中。可能两个版本的文档都会被一个查询匹配到，但被删除的那个旧版本文档在结果集返回前就已经被移除。

###### 3.2.3、保证近实时的搜索

随着按段搜索的发展，一个新的文档从索引到可被搜索的延迟显著降低了，新文档在几分钟之内即可被检索，但这样还是不够快。因为提交一个新的段到磁盘需要一个 fsync 来确保段被物理性的写入磁盘，但是 fsync 的操作代价很大，如果每次索引一个文档都去执行一次的话会造成很大的性能问题。

在 ES 和磁盘之间是文件系统缓存，在内存索引缓冲区中的文档会被写入到一个新的段中，但是这里新段会被写入到文件系统缓存，稍后再被刷新到磁盘，不过只要文件已经在文件系统缓存中，就可以像其他文件一样被打开和读取了。

下面图示，在文件系统缓存中的新段对搜索可见。
![在这里插入图片描述](https://img-blog.csdnimg.cn/55d59a7917794e2699859fdc583f2382.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5ZSJLg==,size_16,color_FFFFFF,t_70,g_se,x_16#pic_center)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2f1b12f99ac841f698220aada5d33d0c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5ZSJLg==,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

> refresh
>
> 在 ES 中，写入和打开一个新段的轻量的过程叫做 refresh 。默认情况下每个分片会每秒自动刷新一次。这就是为什么说 ES 是近实时搜索：文档的变化并不是立即对搜索可见，但是会在一秒之内变为可见。一般情况下每秒会新增一个端。

###### 3.2.4、持久化变更

之前讲到，如果不进行 fsync 操作，数据还是在文件系统缓存中，但系统断电，数据无法恢复，那如何保证数据变更记录在硬盘上。一次完成的提交会将段刷到磁盘，并写入一个包含所有段列表的提交点，ES 在启动或重新打开一个索引的过程中使用这个提交点来判断哪些段属于当前分片。ES 增加了一个 translog，或者叫事务日志，在每一次对 ES 进行非操作时均进行了日志记录，通过 translog，整个流程是下图所示：

1、一个文档被索引之后，就会被添加到内存缓冲区，并且追加到了 translog

![在这里插入图片描述](https://img-blog.csdnimg.cn/fb486a20c48240579659c45ea5a9a1e7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5ZSJLg==,size_16,color_FFFFFF,t_70,g_se,x_16#pic_center)

2、刷新使分片处于下图的状态，分片每秒被刷新一次

​		· 这些在内存缓冲区的文档被写入到一个新的段中，且没有进行 fsync 操作

​		· 这个段被打开，使其可被搜索

​		· 内存缓冲区被清空

![在这里插入图片描述](https://img-blog.csdnimg.cn/2c43b79589a04e0f8a816dddb7d8cc36.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5ZSJLg==,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

3、这个进程继续工作，更多的文档被添加到内存缓冲区和追加到事务日志
![在这里插入图片描述](https://img-blog.csdnimg.cn/726526f63cb34f32b98074eabfbf0467.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5ZSJLg==,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)


4、每隔一段时间，当 translog 变得越来越大时，索引被刷新，一个新的 translog 被创建，并且一个全量提交被执行

- 所有在内存缓冲区的文档都被写入到一个新的段
- 缓存区被清空
- 一个提交点被写入磁盘
- 文件系统缓存通过 fsync 被刷新
- 老的 translog 被删除
![在这里插入图片描述](https://img-blog.csdnimg.cn/41b75c2f4c8d464985457bf6b60c5040.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5ZSJLg==,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)
​	所有在内存中的 segment 被提交到了磁盘，同时清楚 translog

> flush
>
> 在 ES 中，所有在内存中的 segment 被提交到了磁盘，同时清除 translog，一般 Flush 的时间间隔会比较久，默认 30 分钟，或者当 translog 达到一定的大小，也会触发 flush 操作。

ES 的 refresh 操作是为了让最新的数据可以立即被搜索到，而 flush 操作则是为了让数据持久化到磁盘中，另外 ES 的搜索是在内存中处理的，因此 flush 操作不影响数据是否被搜索到。

###### 3.2.5、段合并

之前讲到每次 refresh，一般情况下每秒新增一个段，这样段的数量会比较快的增大。所以后台会进行段合并，小段合并成大段。并且在合并过程中会将旧的已删除文档从文件系统中清除，不会合并到大段中。

合并的过程：

​	1、当索引的时候，刷新操作会创建新的段并将段打开以供搜索使用，此时没有写入磁盘还会保存到磁盘缓存；

​	2、合并过程选择一小部分大小相似的段，并且在后台将它们合并到更大的段中，这并不会中断索引和搜索；

​	3、合并结束，老的段删除，合并完成时的工作：

​		· 新的段被刷新到磁盘中，translog 写入一个包含新段且排除旧的和较小的段的新提交点

​		· 新的段被打开用来搜索

​		· 老的段被删除
![在这里插入图片描述](https://img-blog.csdnimg.cn/319250f5863d4df8a17125eca8ff0c27.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5ZSJLg==,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

##### 3.3、ES并发控制原理

ES 并发写操作，采用的是乐观并发锁控制，由于 ES 中的文档是不可变更的，如果需要更新一个文档，先将当前文档标记为删除，同时新增一个文档，并且将文档 version + 1，目前有两种并发控制：

​	1、内部版本控制：if_seq_no + id_primary_term

​			eg. 首先 put 文档，_seq_no = 0， _primary_term = 1；再次 put，需要根据版本号处理才能更新成功

​	2、外部版本控制(使用外部数据库版本号控制)：version + version_type

​			eg. 首先 put 文档，version = 10，version_type = external；再次 put 文档，必须增大 version，即 version = 11，version_type = external

##### 3.4、原理小结

1. ES 通过多种方式保证搜索的更快，多节点类型，保证了节点的搜索速度，以及故障转移方案
2. 通过倒排索引保证查询的快速，同时由于倒排索引的不变性，更新时需要有效的变更索引的方案
3. 索引变更，是通过新增索引的方式，但是由于可能存在频繁刷入磁盘的操作，导致速度变慢
4. 通过 refresh 操作，新建的段，可被搜索，但是保存在磁盘缓存区
5. 存储在磁盘缓存区，可能会出现断电导致数据丢失的问题，所以数据变更通过 translog 记录下来，当 translog 变的很大时，会进行 flush 操作、translog 写入磁盘
6. refresh 过程导致小段过多，需要进行段合作，小段变大段，同时会清除旧的已删除文档
7. 最后介绍了并发控制的原理，本质上是乐观并发，通过版本号等信息进行控制

