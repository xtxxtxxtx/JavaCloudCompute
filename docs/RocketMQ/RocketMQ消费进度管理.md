﻿业务实现消费回调的时候，当且仅当此回调函数返回ConsumeConcurrentlyStatus.CONSUME_SUCCESS ，RocketMQ才会认为这批消息（默认是1条）是消费完成的

如果这时候消息消费失败，例如数据库异常，余额不足扣款失败等一切业务认为消息需要重试的场景，只要返回ConsumeConcurrentlyStatus.RECONSUME_LATER，RocketMQ就会认为这批消息消费失败了。

为了保证消息是肯定被至少消费成功一次，RocketMQ会把这批消费失败的消息重发回Broker（topic不是原topic而是这个消费租的RETRY topic），在延迟的某个时间点（默认是10秒，业务可设置）后，再次投递到这个ConsumerGroup。而如果一直这样重复消费都持续失败到一定次数（默认16次），就会投递到DLQ死信队列。应用可以监控死信队列来做人工干预。
# 从哪里开始消费
当新的实例启动的时候，PushConsumer会拿到本消费组broker已经记录好的消费进度，如果这个消费进度在broker并没有存储起来，证明这个是一个全新的消费组，此时客户端有几个策略可以选择：
```java
CONSUME_FROM_LAST_OFFSET //默认策略，从该队列最尾开始消费，即跳过历史消息
CONSUME_FROM_FIRST_OFFSET //从队列最开始开始消费，即历史消息（还储存在broker的）全部消费一
遍
CONSUME_FROM_TIMESTAMP//从某个时间点开始消费，和setConsumeTimestamp()配合使用，默认是半
个小时以前
```
# 消息ACK机制
RocketMQ是以Consumer Group+Queue为单位管理消费进度的，以一个Consumer offset标记这个消费组在这条queue上的消费进度。

每次消息成功后，本地的消费进度会被更新，然后由定时器定时同步到broker(即不是立刻同步到broker，有一段时间消费进度只会存在于本地，此时如果宕机那么未提交的消费进度就会被重新消费)，以此来持久化消费进度。但是每次记录消费进度的时候，只会把一批消息中最小的offset值为消费进度值，如下图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/95c77a7c7bd44b928454af5628b1c8bb.png)

> 比如2消费失败，rocketmq跳过2消费到了8,8消费成功，但是提交的时候只会提交消费到1，因为2失败了所以会提交最小成功点。

# 重复消费问题
由于消费进度只是记录了一个下标，就可能出现拉取了100条消息如2101-2200的消息，后面99条都消费结束，只有2101消费一直没有结束的情况。
![在这里插入图片描述](https://img-blog.csdnimg.cn/54c70e1d65154cf0b0af4ac929ddfa93.png)
在这种情况下RocketMQ为了保证消息肯定被消费成功，消费进度只能保持在2101，知道2101也消费结束，本地的消费进度才能标记2200的消费结束（注：consumerOffset=2201）。
此种设计下就有消费大量重复的风险。如2101在还没有消费完成的时候消费实例突然退出（突然断电）。这条queue的消费进度还是维持在2101，当queue重新分配给新的实例的时候，新的实例从broker上拿到的消费进度还是维持在2101，此时就会又从2101开始消费，2102-2200这批消息实际上已经被消费过还是会投递一次。

对于这个场景，RocketMQ暂时无能为力，所以业务必须要保证消息消费的幂等性，这也是RocketMQ官方多次强调的态度。
