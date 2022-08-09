RocketMQ 消息投递分为两种：一种是生产者往MQ Broker中投递；另外一种是MQ Broker往消费者投递(此说法是从消费传递角度阐述，实际上底层是消费者从MQ Broker中Pull拉取的)。
# RocketMQ 消息类型
![请添加图片描述](https://img-blog.csdnimg.cn/9544ae9856aa48468bfa1370e59fe63b.png)
一个Topic可能对应多个实际的消息队列。
在底层实现上，为了提高MQ可用性和灵活性，一个Topic在实际存储过程中采用了多队列的方式，具体形式如上所示。每个消息队列在使用中应当保证先入先出的方式进行消费。

那么存在两个问题：

 - 生产者发送相同topic消息时，消息体应放置在哪一个消息队列中
 - 消费者在消费消息时，该从哪些消息队列中获取消息
# 生产者投递策略
## 轮询算法投递
默认投递方式：基于Queue队列轮询算法投递。
默认情况下采用最简单的轮询算法，优点在于是可以保证每一个queue队列消息投递数量尽可能均匀。算法如下：
```java
/**
*  根据 TopicPublishInfo Topic发布信息对象中维护的index，每次选择队列时，都会递增
*  然后根据 index % queueSize 进行取余，达到轮询的效果
*
*/
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
        return tpInfo.selectOneMessageQueue(lastBrokerName);
}

/**
*  TopicPublishInfo Topic发布信息对象中
*/
public class TopicPublishInfo {
    //基于线程上下文的计数递增，用于轮询目的
    private volatile ThreadLocalIndex sendWhichQueue = new ThreadLocalIndex();
   

    public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
        if (lastBrokerName == null) {
            return selectOneMessageQueue();
        } else {
            int index = this.sendWhichQueue.getAndIncrement();
            for (int i = 0; i < this.messageQueueList.size(); i++) {
                //轮询计算
                int pos = Math.abs(index++) % this.messageQueueList.size();
                if (pos < 0)
                    pos = 0;
                MessageQueue mq = this.messageQueueList.get(pos);
                if (!mq.getBrokerName().equals(lastBrokerName)) {
                    return mq;
                }
            }
            return selectOneMessageQueue();
        }
    }

    public MessageQueue selectOneMessageQueue() {
        int index = this.sendWhichQueue.getAndIncrement();
        int pos = Math.abs(index) % this.messageQueueList.size();
        if (pos < 0)
            pos = 0;
        return this.messageQueueList.get(pos);
    }
}
```
## 消息投递延迟最小策略
默认投递方式加强：基于queue队列轮询算法和消息投递延迟最小的策略投递。
默认投递方式比较简单但是问题在于有些queue队列可能由于自身数量积压等原因，会在投递过程比较长，对于这样的queue队列会影响后续投递效果。
基于这种现象，RocketMQ在每发送一个MQ消息后，都会统计一下消息投递的时间延迟，根据这个时间延迟可以知道往哪些queue投递速度快。
此种场景下会优先使用消息投递延迟最小的策略，如果没有生效再使用queue队列轮询的方式。
```java
public class MQFaultStrategy {
    /**
     * 根据 TopicPublishInfo 内部维护的index,在每次操作时，都会递增，
     * 然后根据 index % queueList.size(),使用了轮询的基础算法
     *
     */
    public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
        if (this.sendLatencyFaultEnable) {
            try {
                // 从queueid 为 0 开始，依次验证broker 是否有效，如果有效
                int index = tpInfo.getSendWhichQueue().getAndIncrement();
                for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
                    //基于index和队列数量取余，确定位置
                    int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
                    if (pos < 0)
                        pos = 0;
                    MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                    if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                        if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
                            return mq;
                    }
                }
                
                // 从延迟容错broker列表中挑选一个容错性最好的一个 broker
                final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
                int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
                if (writeQueueNums > 0) {
                     // 取余挑选其中一个队列
                    final MessageQueue mq = tpInfo.selectOneMessageQueue();
                    if (notBestBroker != null) {
                        mq.setBrokerName(notBestBroker);
                        mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);
                    }
                    return mq;
                } else {
                    latencyFaultTolerance.remove(notBestBroker);
                }
            } catch (Exception e) {
                log.error("Error occurred when selecting message queue", e);
            }
          // 取余挑选其中一个队列
            return tpInfo.selectOneMessageQueue();
        }

        return tpInfo.selectOneMessageQueue(lastBrokerName);
    }
}
```
## 顺序投递策略
上述两种投递方式属于对消息投递的时序性没有要求的场景，这种投递速度和效率比较高。而有些场景下需要保证同类型消息投递和消费的顺序性。
假设现在有topic：topicTest，该topic下有四个queue队列，该topic用于传递订单的状态变迁，假设订单有状态：未支付、已支付、发货中、发货成功、发货失败。

在时序上生产者从时序上可以生成如下几个消息：订单01:未支付——>订单01:已支付——>订单01:发货中——>订单01:发货失败。

消息发送到MQ之后，由于轮询投递原因消息在MQ存储如下：
![请添加图片描述](https://img-blog.csdnimg.cn/36c9ae0678dc4790806ab797198e7235.png)
这种情况下希望消费者消费消息顺序和发送是一致的，然后有上述MQ投递和消费机制，无法保证顺序是正确的。对于顺序异常的消息消费者即使有一定的状态容错，也不能完全处理好这么多随机出现组合的情况。

基于上述情况，RocketMQ 采用的实现方案是：对于相同订单好的消息通过一定的策略将其防止在同一个queue队列中，然后消费者再采用一定的策略(一个线程独立处理一个queue以保证处理消息的顺序性)，最终能保证消费的顺序性。
![请添加图片描述](https://img-blog.csdnimg.cn/de4b955670cd4ff7853fb2047ada5911.png)
生产者是如何将相同订单号的消息发送到同一个queue队列中的：生产者在消息投递过程中，使用MessageQueueSelector作为队列选择的策略接口，定义如下：
```java
public interface MessageQueueSelector {
        /**
         * 根据消息体和参数，从一批消息队列中挑选出一个合适的消息队列
         * @param mqs  待选择的MQ队列选择列表
         * @param msg  待发送的消息体
         * @param arg  附加参数
         * @return  选择后的队列
         */
        MessageQueue select(final List<MessageQueue> mqs, final Message msg, final Object arg);
}
```
RocketMQ 提供如下几种实现：
![在这里插入图片描述](https://img-blog.csdnimg.cn/8965c19a1741448d872151cb7f4ad3fb.png)
|投递策略|策略实现类|说明|
|--|--|--|
| 随机分配策略|SelectMessageQueueByRandom|简单的随机数选择算法|
| 基于Hash分配策略|SelectMessageQueueByHash|根据附加参数计算Hash值，按照消息队列列表大小取余数，得到消息队列的index|
| 基于机器机房位置分配策略|SelectMessageQueueByMachineRandom|开源版本无具体实现，基本目的是机器就近原则分配|

基于Hash分配策略源码：
```java
public class SelectMessageQueueByHash implements MessageQueueSelector {

    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        int value = arg.hashCode();
        if (value < 0) {
            value = Math.abs(value);
        }

        value = value % mqs.size();
        return mqs.get(value);
    }
}
```
# 消费者分配策略
RocketMQ 对于消费者消费消息有两种形式：

 - BROADCASTING：广播式消费，该模式下一个消息会被通知到每一个消费者
 - CLUSTERING：集群式消费，该模式下一个消息最多只会被投递到一个消费者上进行消费

如下所示：
![请添加图片描述](https://img-blog.csdnimg.cn/45708508e8204da18154c6dc99a139ad.png)
广播模式比较简单，主要说一下集群式消费。
对于使用了消费模式为MessageModel.CLUSTERING进行消费时，需要保证一个消息在整个集群中只需要被消费一次。实际上在RocketMQ底层，消息指定分配给消费者的实现是通过queue队列分配给消费者方式完成的。也就是说消息分配单位是消息所在的queue队列。即：将queue队列指定给特定的消费者后，queue队列内的所有消息将会被指定到消费者进行消费。

RocketMQ中定义了策略接口AllocateMessageQueueStrategy，对于给定的消费者分组和消息队列列表、消费者列表，当前消费者应当被分配到哪些queue队列，定义如下：
```java
/**
 * 为消费者分配queue的策略算法接口
 */
public interface AllocateMessageQueueStrategy {

    /**
     * Allocating by consumer id
     *
     * @param consumerGroup 当前 consumer群组
     * @param currentCID 当前consumer id
     * @param mqAll 当前topic的所有queue实例引用
     * @param cidAll 当前 consumer群组下所有的consumer id set集合
     * @return 根据策略给当前consumer分配的queue列表
     */
    List<MessageQueue> allocate(
        final String consumerGroup,
        final String currentCID,
        final List<MessageQueue> mqAll,
        final List<String> cidAll
    );

    /**
     * 算法名称
     *
     * @return The strategy name
     */
    String getName();
}
```
RocketMQ 提供如下几种实现：
![在这里插入图片描述](https://img-blog.csdnimg.cn/dedb3b5b0ffa454791cb4e53e8c7b327.png)
|算法名称|含义|
|--|--|
| AllocateMessageQueueAveragely | 平均分配算法|
|  AllocateMessageQueueAveragelyByCircle| 基于环形平均分配算法 |
| AllocateMachineRoomNearby | 基于机房临近原则算法 |
| AllocateMessageQueueByMachineRoom | 基于机房分配算法 |
| AllocateMessageQueueConsistentHash | 基于一致性hash算法 |
| AllocateMessageQueueByConfig | 基于配置分配算法 |

eg.假设当前同一个topic下有queue队列10个，消费者共有4个。
使用方式：默认消费者使用使用了 AllocateMessageQueueAveragely平均分配策略。如果需要使用其他分配策略，使用方式如下：
```java
//创建一个消息消费者，并设置一个消息消费者组，并指定使用一致性hash算法的分配策略
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(null,"rocket_test_consumer_group",null,new AllocateMessageQueueConsistentHash());
.....
```
## 平均分配算法
平均分配算法并不是指严格意义上的完全平均，如上例10个queue，消费者4个，无法进行整除。除整除之外多出来的queue，将依次根据消费者的顺序均摊。

按照上述例子来看，即每个消费者均摊2个queue，除了均摊之外，多出来2个queue还没有分配，那么，根据消费者的顺序consumer-1、consumer-2、consumer-3、consumer-4,则多出来的2个queue将分别给consumer-1和consumer-2。
最终分摊关系如下：

 - Consumer-1:3个
 - Consumer-2:3个
 - Consumer-3:2个
 - Consumer-4:2个

代码实现较简单：
```java
public class AllocateMessageQueueAveragely implements AllocateMessageQueueStrategy {
    private final InternalLogger log = ClientLogger.getLog();

    @Override
    public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
        List<String> cidAll) {
        if (currentCID == null || currentCID.length() < 1) {
            throw new IllegalArgumentException("currentCID is empty");
        }
        if (mqAll == null || mqAll.isEmpty()) {
            throw new IllegalArgumentException("mqAll is null or mqAll empty");
        }
        if (cidAll == null || cidAll.isEmpty()) {
            throw new IllegalArgumentException("cidAll is null or cidAll empty");
        }

        List<MessageQueue> result = new ArrayList<MessageQueue>();
        if (!cidAll.contains(currentCID)) {
            log.info("[BUG] ConsumerGroup: {} The consumerId: {} not in cidAll: {}",
                consumerGroup,
                currentCID,
                cidAll);
            return result;
        }

        int index = cidAll.indexOf(currentCID);
        int mod = mqAll.size() % cidAll.size();
        int averageSize =
            mqAll.size() <= cidAll.size() ? 1 : (mod > 0 && index < mod ? mqAll.size() / cidAll.size()
                + 1 : mqAll.size() / cidAll.size());
        int startIndex = (mod > 0 && index < mod) ? index * averageSize : index * averageSize + mod;
        int range = Math.min(averageSize, mqAll.size() - startIndex);
        for (int i = 0; i < range; i++) {
            result.add(mqAll.get((startIndex + i) % mqAll.size()));
        }
        return result;
    }

    @Override
    public String getName() {
        return "AVG";
    }
}
```
## 基于环形平均算法
环形平均算法，是指根据消费者的顺序，依次在由queue队列组成的环形图中逐个分配。具体流程如下所示:
![在这里插入图片描述](https://img-blog.csdnimg.cn/34103b1f7fbe41c6b24abc873a4d7ca9.png)
最终分配结果是：

 - Consumer-1:0、4、8
 - Consumer-2:1、5、9
 - Consumer-3:2、6
 - Consumer-4:3、7

代码实现如下：
```java
public class AllocateMessageQueueAveragelyByCircle implements AllocateMessageQueueStrategy {
    private final InternalLogger log = ClientLogger.getLog();

    @Override
    public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
        List<String> cidAll) {
        if (currentCID == null || currentCID.length() < 1) {
            throw new IllegalArgumentException("currentCID is empty");
        }
        if (mqAll == null || mqAll.isEmpty()) {
            throw new IllegalArgumentException("mqAll is null or mqAll empty");
        }
        if (cidAll == null || cidAll.isEmpty()) {
            throw new IllegalArgumentException("cidAll is null or cidAll empty");
        }

        List<MessageQueue> result = new ArrayList<MessageQueue>();
        if (!cidAll.contains(currentCID)) {
            log.info("[BUG] ConsumerGroup: {} The consumerId: {} not in cidAll: {}",
                consumerGroup,
                currentCID,
                cidAll);
            return result;
        }

        int index = cidAll.indexOf(currentCID);
        for (int i = index; i < mqAll.size(); i++) {
            if (i % cidAll.size() == index) {
                result.add(mqAll.get(i));
            }
        }
        return result;
    }

    @Override
    public String getName() {
        return "AVG_BY_CIRCLE";
    }
}
```
## 一致性hash分配算法
该算法会将consumer消费者作为Node节点构造成一个hash环，然后queue队列通过这个hash环来决定被分配给哪个consumer消费者。
![在这里插入图片描述](https://img-blog.csdnimg.cn/7cba399ee37249159cd931969c9c2c0c.png)
一致性hash算法用在分布式系统中，保证数据一致性而提出的一种基于hash环实现的算法：
```java
public class AllocateMessageQueueConsistentHash implements AllocateMessageQueueStrategy {
    private final InternalLogger log = ClientLogger.getLog();

    private final int virtualNodeCnt;
    private final HashFunction customHashFunction;

    public AllocateMessageQueueConsistentHash() {
        this(10);
    }

    public AllocateMessageQueueConsistentHash(int virtualNodeCnt) {
        this(virtualNodeCnt, null);
    }

    public AllocateMessageQueueConsistentHash(int virtualNodeCnt, HashFunction customHashFunction) {
        if (virtualNodeCnt < 0) {
            throw new IllegalArgumentException("illegal virtualNodeCnt :" + virtualNodeCnt);
        }
        this.virtualNodeCnt = virtualNodeCnt;
        this.customHashFunction = customHashFunction;
    }

    @Override
    public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
        List<String> cidAll) {

        if (currentCID == null || currentCID.length() < 1) {
            throw new IllegalArgumentException("currentCID is empty");
        }
        if (mqAll == null || mqAll.isEmpty()) {
            throw new IllegalArgumentException("mqAll is null or mqAll empty");
        }
        if (cidAll == null || cidAll.isEmpty()) {
            throw new IllegalArgumentException("cidAll is null or cidAll empty");
        }

        List<MessageQueue> result = new ArrayList<MessageQueue>();
        if (!cidAll.contains(currentCID)) {
            log.info("[BUG] ConsumerGroup: {} The consumerId: {} not in cidAll: {}",
                consumerGroup,
                currentCID,
                cidAll);
            return result;
        }

        Collection<ClientNode> cidNodes = new ArrayList<ClientNode>();
        for (String cid : cidAll) {
            cidNodes.add(new ClientNode(cid));
        }

        final ConsistentHashRouter<ClientNode> router; //for building hash ring
        if (customHashFunction != null) {
            router = new ConsistentHashRouter<ClientNode>(cidNodes, virtualNodeCnt, customHashFunction);
        } else {
            router = new ConsistentHashRouter<ClientNode>(cidNodes, virtualNodeCnt);
        }

        List<MessageQueue> results = new ArrayList<MessageQueue>();
        for (MessageQueue mq : mqAll) {
            ClientNode clientNode = router.routeNode(mq.toString());
            if (clientNode != null && currentCID.equals(clientNode.getKey())) {
                results.add(mq);
            }
        }

        return results;

    }

    @Override
    public String getName() {
        return "CONSISTENT_HASH";
    }

    private static class ClientNode implements Node {
        private final String clientID;

        public ClientNode(String clientID) {
            this.clientID = clientID;
        }

        @Override
        public String getKey() {
            return clientID;
        }
    }
}
```
## 机房临近分配算法
该算法使用装饰者设计模式对分配策略进行增强，一般在生产环境如果是微服务架构下，RocketMQ 集群部署可能是在不同的机房部署其基本结构如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/258502fa1dd44c5392c779e085c64ad6.png)
对于跨机房的场景会存在网络、稳定性和隔离心的原因，该算法会根据queue的部署机房位置和消费者consumer的位置，过滤出当前消费者consumer相同机房的queue队列，然后再结合上述的算法如基于平均分配算法在queue队列子集的基础上再挑选。相关代码实现如下：
```java
 public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
        List<String> cidAll) {
        if (currentCID == null || currentCID.length() < 1) {
            throw new IllegalArgumentException("currentCID is empty");
        }
        if (mqAll == null || mqAll.isEmpty()) {
            throw new IllegalArgumentException("mqAll is null or mqAll empty");
        }
        if (cidAll == null || cidAll.isEmpty()) {
            throw new IllegalArgumentException("cidAll is null or cidAll empty");
        }

        List<MessageQueue> result = new ArrayList<MessageQueue>();
        if (!cidAll.contains(currentCID)) {
            log.info("[BUG] ConsumerGroup: {} The consumerId: {} not in cidAll: {}",
                consumerGroup,
                currentCID,
                cidAll);
            return result;
        }

        //group mq by machine room
        Map<String/*machine room */, List<MessageQueue>> mr2Mq = new TreeMap<String, List<MessageQueue>>();
        for (MessageQueue mq : mqAll) {
            String brokerMachineRoom = machineRoomResolver.brokerDeployIn(mq);
            if (StringUtils.isNoneEmpty(brokerMachineRoom)) {
                if (mr2Mq.get(brokerMachineRoom) == null) {
                    mr2Mq.put(brokerMachineRoom, new ArrayList<MessageQueue>());
                }
                mr2Mq.get(brokerMachineRoom).add(mq);
            } else {
                throw new IllegalArgumentException("Machine room is null for mq " + mq);
            }
        }

        //group consumer by machine room
        Map<String/*machine room */, List<String/*clientId*/>> mr2c = new TreeMap<String, List<String>>();
        for (String cid : cidAll) {
            String consumerMachineRoom = machineRoomResolver.consumerDeployIn(cid);
            if (StringUtils.isNoneEmpty(consumerMachineRoom)) {
                if (mr2c.get(consumerMachineRoom) == null) {
                    mr2c.put(consumerMachineRoom, new ArrayList<String>());
                }
                mr2c.get(consumerMachineRoom).add(cid);
            } else {
                throw new IllegalArgumentException("Machine room is null for consumer id " + cid);
            }
        }

        List<MessageQueue> allocateResults = new ArrayList<MessageQueue>();

        //1.allocate the mq that deploy in the same machine room with the current consumer
        String currentMachineRoom = machineRoomResolver.consumerDeployIn(currentCID);
        List<MessageQueue> mqInThisMachineRoom = mr2Mq.remove(currentMachineRoom);
        List<String> consumerInThisMachineRoom = mr2c.get(currentMachineRoom);
        if (mqInThisMachineRoom != null && !mqInThisMachineRoom.isEmpty()) {
            allocateResults.addAll(allocateMessageQueueStrategy.allocate(consumerGroup, currentCID, mqInThisMachineRoom, consumerInThisMachineRoom));
        }

        //2.allocate the rest mq to each machine room if there are no consumer alive in that machine room
        for (String machineRoom : mr2Mq.keySet()) {
            if (!mr2c.containsKey(machineRoom)) { // no alive consumer in the corresponding machine room, so all consumers share these queues
                allocateResults.addAll(allocateMessageQueueStrategy.allocate(consumerGroup, currentCID, mr2Mq.get(machineRoom), cidAll));
            }
        }

        return allocateResults;
    }
```
## 基于机房分配算法
该算法适用于属于同一个机房内部的消息去分配queue。此方式非常明确基于上面的机房临近分配算法的场景，这种更彻底直接指定基于机房消费的策略。此方式具有强约定性比如broker名称按照机房的名称进行拼接，在算法中通过约定解析进行分配。
```java
/**
 * Computer room Hashing queue algorithm, such as Alipay logic room
 */
public class AllocateMessageQueueByMachineRoom implements AllocateMessageQueueStrategy {
    private Set<String> consumeridcs;

    @Override
    public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
        List<String> cidAll) {
        List<MessageQueue> result = new ArrayList<MessageQueue>();
        int currentIndex = cidAll.indexOf(currentCID);
        if (currentIndex < 0) {
            return result;
        }
        List<MessageQueue> premqAll = new ArrayList<MessageQueue>();
        for (MessageQueue mq : mqAll) {
            String[] temp = mq.getBrokerName().split("@");
            if (temp.length == 2 && consumeridcs.contains(temp[0])) {
                premqAll.add(mq);
            }
        }

        int mod = premqAll.size() / cidAll.size();
        int rem = premqAll.size() % cidAll.size();
        int startIndex = mod * currentIndex;
        int endIndex = startIndex + mod;
        for (int i = startIndex; i < endIndex; i++) {
            result.add(mqAll.get(i));
        }
        if (rem > currentIndex) {
            result.add(premqAll.get(currentIndex + mod * cidAll.size()));
        }
        return result;
    }

    @Override
    public String getName() {
        return "MACHINE_ROOM";
    }

    public Set<String> getConsumeridcs() {
        return consumeridcs;
    }

    public void setConsumeridcs(Set<String> consumeridcs) {
        this.consumeridcs = consumeridcs;
    }
}
```
