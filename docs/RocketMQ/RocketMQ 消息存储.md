目前的MQ中间件从存储模型来，分为需要持久化和不需要持久化的两种模型。现在大多数的是支持持久化存储的，比如ActiveMQ、RabbitMQ、Kafka、RocketMQ。ZeroMQ却不需要支持持久化存储而业务系统也大多需要MQ有持久存储的能力，这样可以大大增加系统的高可用性。

从存储方式和效率来看，文件系统高于KV存储，KV存储又高于关系型数据库，直接操作文件系统肯定是最快的，但如果从可靠性的角度出发直接操作文件系统是最低的，而关系型数据库的可靠性是最高的。
# 存储介质类型对比
常用的存储类型分为关系型数据库存储、分布式KV存储和文件系统存储。

目前的高性能磁盘，顺序写速度可以达到600MB/s，足以满足一般网卡的传输速度，而磁盘随机读写速度只有100KB/s，与顺序写性能差了6000倍。因此好的消息队列系统都会采用顺序写的方式。

|  | 关系型数据库存储| 分布式kv存储 | 文件系统存储 |
|--|--|--|--|
| 简介 | 选用jdbc方式实现消息持久化，只需要简单配置xml即可实现jdbc消息存储| kv存储即key-value型存储中间件| 将消息存储到文件系统中 |
| 性能 | 存在性能瓶颈，如mysql在单表数据量达到千万级别IO读写性能下降| 通过高并发中间件存储和处理消息，速度必然优于数据库存储方式 | 将消息刷盘至所部属虚拟化/物理机的文件系统来实现消息持久化，效率更高 |
| 可靠性 | 该方案十分依赖DB，一旦DB出现故障，MQ消息无法落盘存储从而导致线上事故| 相较DB来说更加安全可靠| 除非部署MQ的机器本身或是本地磁盘挂了否则一般不会出现无法持久化的问题 |
| 项目使用 | ActiveMQ| Redis、RockDB| RocketMQ、Kafka、RabbitMQ |

存储效率：文件系统 > 分布式kv存储 > 关系型数据库DB
开发难度和集成：关系型数据库DB < 分布式kv存储 < 文件系统
# 顺序读写和随机读写
|  | 顺序读写| 随机读写 | 
|--|--|--|
| 文件数目 | 读取一个大文件| 读取多个小文件| 
| 比较 | 顺序读写只读取一个大文件，耗时更少| 随机读写需要打开多个文件，写进行多次的训导和旋转延迟，标绿远低于顺序读写 | 
| 文件预读 | 顺序读写时磁盘会预读文件，即在读取的起始地址连续读取多个页面，若被预读的页面被使用则无需再去读取| 由于数据不在一起，无法预读| 
| 比较 | 在大并发情况下磁盘预读能免去大量读操作，处理速度肯定更快| 磁盘需要不断的寻址，效率很低|
| 写入数据 | 写入新文件时需要寻找磁盘可用空间| 写入新文件时需要寻找磁盘可用空间。但由于一个文件的存储量更小，这个操作触发频率更多| 
| 比较 | 顺序读写创建新文件，只需要创建一个大文件就可使用很久| 随机读写肯频繁创建文件。创建文件时需要进行寻找磁盘可用空间等一系列操作，肯定更加耗时|
# 消息存储设计
RocketMQ 采用单一的日志文件，即把同一台机器上面所有topic的所有queue消息，存放在一个文件里面，从而避免了随机的磁盘写入。
![请添加图片描述](https://img-blog.csdnimg.cn/2fa34c8854ce4bada20ea6074b448c89.png)
所有消息都存在一个单一的CommitLog文件里面，然后有后台线程异步的同步到ConsumerQueue，再有Consumer进行消费。
这里之所以可以用异步线程，也是因为消息队列天生就是用来缓冲消息的。只要消息到CommitLog发送的消息也就不会丢失。只要消息不丢，那就有了充足的回旋余地，用一个后台线程慢慢同步到ConsumerQueue，再由Consumer消费。
# 消息存储结构
消息主体以及元数据的存储主体，存储Producer端写入的消息主体内容，消息内容不是定长的。RocketMQ采取一定机制，尽量向CommitLog中顺序写，但是随机读。单个文件大小默认1G，可通过在broker文件中设置mapedFileSizeCommitLog属性来改变默认大小。

 - CommitLog：消息存储文件，所有消息主题的消息都存储在CommitLog文件中
 - ConsumeQueue：消息消费队列，消息到达CommitLog文件后，将异步抓发到消息消费队列，供消息消费者消费
 - indexFile：消息索引文件，主要存储消息Key与Offset的对应关系
## CommitLog
CommitLog以物理文件的方式存放，每台Broker上的CommitLog被本机器所有的ConsumeQueue共享，文件地址：$ {user.home} \store$ { commitlog} \ $ { fileName}

文件名长度为20位，左边补零，剩余为起始偏移量，比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1073741824；当第一个文件写满了，第二个文件为00000000001073741824，起始偏移量为1073741824，以此类推。消息主要是顺序写入日志文件，当文件满了，写入下一个文件。

Commitlog 文件存储的逻辑视图如下，每条消息的前面 4 个字节存储该条消息的总长度。但是一个消息的存储长度是不固定的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/f6bcc5d2838e49fd82c9bd1e29eed5dc.png)
文件的消息单元存储详细信息：
![在这里插入图片描述](https://img-blog.csdnimg.cn/108d753718a94dbd9da997eae2c1d002.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/4f4ac99c374f442ca6955b6187ab5f5b.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/21c42f6c69b54524bfac6f7f9839983b.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/223a9bfb1a674b25a3cb5b8fd811aa95.png)
## ConsumeQueue
RocketMQ 基于主题订阅模式实现消息的消费，消费者关系的是主题下的所有消息。但是由于不同的主题消息不连续的存储在CommitLog文件中，如果只是检索该消息文件可想而知会有多慢，为了提高效率对应的主题队列建立了索引文件，为了加快消息的检索和节省磁盘空间，每一个ConsumeQueue条目存储了消息的关键信息CommitLog文件中的偏移量、消息长度、tag的hashcode值。
![请添加图片描述](https://img-blog.csdnimg.cn/eaac167d7a844806b046b020e67f2372.png)
ConsumeQueue是消息的逻辑队列，类似数据库的索引文件。存储的是指向物理存储的地址。每个Topic下的每个Message Queue都有一个对应的ConsumeQueue文件，文件地址在$ {$storeRoot} \consumequeue$ {topicName} $ { queueld} $ {fileName}。

单个consumequeue文件中默认包含30万个条目，每个条目20个字节，所以每个文件的大小是固定的30w x 20字节，单个consumequeue文件可认为是一个数组，下标即为逻辑偏移量，消息的消费进度存储的偏移量即逻辑偏移量。


ConsumeQueue中并不需要存储消息的内容，而存储的是消息在CommitLog中的offset。也就是说ConsumeQueue其实是CommitLog的一个索引文件。
![在这里插入图片描述](https://img-blog.csdnimg.cn/70718c5982734d2fb6a6a9e95db659a2.png)
ConsumeQueue是定长的结构，每1条记录固定的20个字节。很显然，Consumer消费消息的时候，要读2次：先读ConsumeQueue得到offset，再读CommitLog得到消息内容。

> 简单来说就是消费消息的时候consumer向 ConsumeQueue发送消费申请，也就是读取内容，得到消息的offset，再拿着offset去CommitLog读取到真正的消息内容


​ ConsumeQueue 即为 Commitlog 文件的索引文件，其构建机制是 当消息到达 Commitlog 文件后由专门的线程产生消息转发任务，从而构建消息消费队列文件（ConsumeQueue ）与下文提到的索引文件。

### 设计的优势
存储机制如此设计的优势：

 - CommitLog顺序写，可以大幅度的提高写入效率
 - 虽然是随机读，但是利用操作系统的pagecache机制，可以批量的从磁盘读取，作为cache存到内存中，加速后续的读取速度
 - 为了保证完全的顺序写，需要ConsumeQueue这个中间结构，因为ConsumeQueue里面只存储偏移量信息，所以尺寸是有限的，在实际情况中大部分的ConsumeQueue能够被全部读入内存，所以这个中间结构的操作速度很快，可以认为是内存读取的速度。此外为了保证CommitLog和ConsumeQueue的一致性，CommitLog里存储了ConsumeQueues、Message Key、Tag等所有信息，即使ConsumeQueue丢失也可以通过CommitLog完全恢复出来。
## IndexFile
index存储的是索引文件，用于为生成的索引文件提供访问服务，这个文件用来加快消息查询的速度，通过消息key值查询消息真正的实体内容。消息消费队列RocketMQ专门为消息订阅构建的索引文件，提高根据主题与消息检索消息的速度，使用Hash索引机制，具体是Hash槽与Hash冲突的链表结构。

在实际的物理存储上，文件名则是以创建时的时间戳命名的，固定的单个indexFile文件大小约400M，一个indexFile可以保存2000W个索引。
### IndexFile结构分析
![请添加图片描述](https://img-blog.csdnimg.cn/fec32fdc2a2646cc8ecc86c792f4a7ae.png)
#### IndexHead数据：
 - beginTimeStamp：该索引文件包含消息的最小存储时间
 - endTimeStamp：该索引文件包含消息的最大存储时间
 - beginPhyoffset：该索引文件中包含消息的最小物理偏移量（Commitlog文件偏移量）
 - endPhyoffset：该索引文件中包含消息的最大物理偏移量（Commitlog文件偏移量）
 - hashSlotCount：hashslot个数，并不是hash槽使用的个数，在这里意义不大
 - indexCount：已使用的index条目个数
#### Hash槽
​ 一个 IndexFile 默认包含 500W 个 Hash 槽，每个 Hash 槽存储的是落在该 Hash 槽的 hashcode 最新的 Index 的索引
#### index条目列表
 - hashcode：key的hashcode
 - phyoffset：消息对应的物理偏移量
 - timeoff：该消息存储时间与第一条消息的时间戳的差值，小于0表示该消息无效
 - preIndexNo：该条目的前一条记录的Index索引，hash冲突时根据该值构建链表结构

#### 通过key查找消息
![请添加图片描述](https://img-blog.csdnimg.cn/36d889022a80458b839e3affd62526d1.png)
```java
public void selectPhyOffset(final List<Long> phyOffsets, final String key, final int maxNum,
	final long begin, final long end, boolean lock) {
	if (this.mappedFile.hold()) {
		//获取key的hash信息
		int keyHash = indexKeyHashMethod(key);
		//获取hash槽的下标
		int slotPos = keyHash % this.hashSlotNum;
		//获取hash槽的物理地址
		int absSlotPos = IndexHeader.INDEX_HEADER_SIZE + slotPos * hashSlotSize;
		FileLock fileLock = null;
		try {
			if (lock) {
				// fileLock = this.fileChannel.lock(absSlotPos,
				// hashSlotSize, true);
			}
			//获取hash槽的值
			int slotValue = this.mappedByteBuffer.getInt(absSlotPos);
			// if (fileLock != null) {
			// fileLock.release();
			// fileLock = null;
			// }
			//判断值是否小于等于0或者 大于当前索引文件的最大条目
			if (slotValue <= invalidIndex || slotValue > this.indexHeader.getIndexCount()
				|| this.indexHeader.getIndexCount() <= 1) {
			} else {
				for (int nextIndexToRead = slotValue; ; ) {
					if (phyOffsets.size() >= maxNum) {
						break;
					}
					//计算条目的物理地址  = 索引头部大小（40字节） + hash槽的大小(4字节)*槽的数量（500w） + 当前索引最大条目的个数*每index的大小（20字节）
					int absIndexPos =
						IndexHeader.INDEX_HEADER_SIZE + this.hashSlotNum * hashSlotSize
							+ nextIndexToRead * indexSize;
					//获取key的hash值
					int keyHashRead = this.mappedByteBuffer.getInt(absIndexPos);
					//获取消息的物理偏移量
					long phyOffsetRead = this.mappedByteBuffer.getLong(absIndexPos + 4);
					//获取当前消息的存储时间戳与index文件的时间戳差值
					long timeDiff = (long) this.mappedByteBuffer.getInt(absIndexPos + 4 + 8);
					//获取前一个条目的信息（链表结构）
					int prevIndexRead = this.mappedByteBuffer.getInt(absIndexPos + 4 + 8 + 4);
					if (timeDiff < 0) {
						break;
					}
					timeDiff *= 1000L;
					long timeRead = this.indexHeader.getBeginTimestamp() + timeDiff;
					//判断该消息是否在查询的区间
					boolean timeMatched = (timeRead >= begin) && (timeRead <= end);
					//判断key的hash值是否相等并且在查询的时间区间内
					if (keyHash == keyHashRead && timeMatched) {
						//加入到物理偏移量的List中
						phyOffsets.add(phyOffsetRead);
					}
					if (prevIndexRead <= invalidIndex
						|| prevIndexRead > this.indexHeader.getIndexCount()
						|| prevIndexRead == nextIndexToRead || timeRead < begin) {
						break;
					}
					//继续前一个条目信息获取进行匹配
					nextIndexToRead = prevIndexRead;
				}
			}
		} catch (Exception e) {
			log.error("selectPhyOffset exception ", e);
		} finally {
			if (fileLock != null) {
				try {
					fileLock.release();
				} catch (IOException e) {
					log.error("Failed to release the lock", e);
				}
			}
			this.mappedFile.release();
		}
	}
}
```
 1. 根据查询的 key 的 hashcode%slotNum 得到具体的槽的位置（ slotNum 是一个索引文件里面包含的最大槽的数目，例如图中所示slotNum=5000000）
 2. 根据 slotValue（ slot 位置对应的值）查找到索引项列表的最后一项（倒序排列， slotValue 总是指向最新的一个 索引项）
 3. 遍历索引项列表返回查询时间范围内的结果集（默认一次最大返回的 32 条记彔）
 4. Hash 冲突：寻找 key 的 slot 位置时相当于执行了两次散列函数，一次key的hash，一次key的hash值取模，因此返里存在两次冲突的情况；第一种， key 的 hash 不同但模数相同，此时查询的时候会在比较一次key 的hash 值（每个索引项保存了 key 的 hash 值），过滤掉 hash 值不相等的项。第二种， hash 值相等但 key 不等，出于性能的考虑冲突的检测放到客户端处理（ key 的原始值是存储在消息文件中的，避免对数据文件的解析），客户端比较一次消息体的 key 是否相同
#### checkpoint
checkpoint文件的作用是记录CommitLog、ConsumeQueue、index文件的刷盘时间点。
文件固定长度4k，其中只用了该文件的前24个字节，查看其存储格式：
![请添加图片描述](https://img-blog.csdnimg.cn/940d3745cc9d47cd98d0725c8230ec20.png)
 - physicMsgTimestamp：commitlog文件刷盘时间点
 - logicsMsgTimestamp：消息的消费队列文件刷盘时间点
 - indexMsgTimestamp：索引文件刷盘时间点
### Config
config文件夹中存储着Topic和Consumer等相关信息。主题和消费者群组相关的信息就存在在此。
 - topics.json：topic配置属性
 - subscriptionGroup.json：延时消息队列拉取进度
 - consumerOffset.json：集群消费模式消息消费进度
 - consumerFilter.json：主题消息过滤信息
#### abort
如果存在abort文件说明Broker非正常关闭，该文件默认启动时创建，正常退出之前删除。
