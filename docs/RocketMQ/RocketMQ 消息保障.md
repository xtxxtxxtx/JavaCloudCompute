# 1. 生产端保障
生产端保障需要从以下几个方面来保障：

 - 使用可靠的消息发送方式
 - 注意生产端重试
 - 生产禁止自动创建Topic

## 1.1 消息发送保障
### 1.1.1 同步发送
发送者向MQ执行发送消息API时，同步等待直到消息服务器返回发送结果，会在收到接收方发回响应之后才发下一个数据包的通讯方式，该方式只有在消息完全发送完成之后才返回结果，此方式存在需要同步等待发送结果的时间代价。
![请添加图片描述](https://img-blog.csdnimg.cn/c05d694a929b429686d00de473f17d33.png)
简单说同步发送就是指Producer发送消息后，会在接收到broker响应后才继续发下一条消息的通信方式。

使用场景：由于这种同步发送方式确保消息的可靠性，同时也能及时得到消息发送的结果，因此适合一些发送比较重要的消息场景，比如说重要的通知邮件。

注意事项：此方式具有内部重试机制，即在主动声明本次消息发送失败之前内部实现将重试一定次数，默认为2次(DefaultMQProducer#getRetryTimesWhenSendFailed)。存在同一个消息可能被多次发送给broker的问题，这里需要应用的开发者自己在消费端处理幂等问题。
### 1.1.2 异步发送
异步发送是指发送方发出数据后不等接收方发回响应，接着发送下个数据报的通讯方式。MQ异步发送需要用户实现异步发送回调接口。
![请添加图片描述](https://img-blog.csdnimg.cn/3548cef3c18b44058e702cc32903560d.png)
异步发送是指Producer发出一条消息后不需要等broker响应，就接着发送下一条消息的通信方式。需要注意的是不等待broker响应并不意味着broker不响应，而是通过回调接口来接收broker的响应。因此要记住一点异步发送同样可以对消息的响应结果进行处理。

使用场景：由于异步发送不需要等待broker响应，因此在一些比较注重RD的场景就会比较适用。比如在一些视频上传场景。

注意事项：RocketMQ内部只对同步模式进行了重试，异步发送模式是没有自动重试的需要手动实现。
### 1.1.3 单向发送
较简单，就是只管发不管有没有抵达。
## 1.2 消息发送总结
| 发送方式| 发送TPS| 发送结果反馈| 可靠性| 适用场景|
|--|--|--|--|--|
| 同步发送 | 一般 | 有 | 不丢失 | 重要的通知场景 |
| 异步发送 | 快 | 有 | 不丢失 | 比较注重RT的场景 |
| 单向发送 | 最快 | 无 | 可能丢失 | 可靠性要求并不高的场景 |
## 1.3 发送状态
发送消息时，将获得包含SendStatus的SendResult。首先，我们假设Message的isWaitStoreMsgOK = true（默认为true），如果没有抛出异常将始终获得SEND_OK，以下是每个状态的说明列表：

**FLUSH_DISK_TIMEOUT**
如果设置 FlushDiskType=SYNC_FLUSH (默认是 ASYNC_FLUSH)，并且 Broker 没有在syncFlushTimeout（默认是 5 秒）设置的时间内完成刷盘，就会收到此状态码。

**FLUSH_SLAVE_TIMEOUT**
如果设置为 SYNC_MASTER ，并且 slave Broker 没有在 syncFlushTimeout 设定时间内完成同步，就会收到此状态码。

**SLAVE_NOT_AVAILABLE**
如果设置为SYNC_MASTER ，并没有配置 slave Broker，就会收到此状态码。

**SEND_OK**
这个状态可以简单理解为，没有发生上面列出的三个问题状态就是SEND_OK。需要注意的是，SEND_OK并不意味着可靠，如果想严格确保没有消息丢失，需要开启SYNC_MASTER orSYNC_FLUSH。
## 1.4 MQ 发送端重试保障
如果由于网络抖动等原因，Producer程序向Broker发送消息时没有成功，即发送端没有收到Broker的ACK。导致最终Consumer无法消费消息，此时RocketMQ会自动进行重试。

DefaultMQProducer可以设置消息发送失败的最大重试次数，并可以结合发送的超时时间来进行重试的处理，具体API如下：
```java
//设置消息发送失败时的最大重试次数
public void setRetryTimesWhenSendFailed(int retryTimesWhenSendFailed) {
	this.retryTimesWhenSendFailed = retryTimesWhenSendFailed;
}
//同步发送消息，并指定超时时间
public SendResult send(Message msg, long timeout) throws MQClientException, RemotingException,
MQBrokerException, InterruptedException {
	return this.defaultMQProducerImpl.send(msg, timeout);
```
### 1.4.1 重试问题
超时重试针对网上说的超时异常会重试的说法大部分是错误的。

> 是因为下面测试代码的超时时间设置为5毫秒 ，按照正常肯定会报超时异常，但设置1次重试和3000次的重试，虽然最终都会报下面异常，但输出错误时间报显然不应该是一个级别。但测试发现无论设置的多少次的重试次数，报异常的时间都差不多。按道理说重试次数越多，报异常的时间跨度应该越大

原因：查看源码发现只有同步发送才会重试，并且超时是不重试的。
```java
/**
* 说明 抽取部分代码
*/
private SendResult sendDefaultImpl(Message msg, final CommunicationMode
communicationMode, final SendCallback sendCallback, final long timeout) {
	//1、获取当前时间
	long beginTimestampFirst = System.currentTimeMillis();
	long beginTimestampPrev ;
	//2、去服务器看下有没有主题消息
	TopicPublishInfo topicPublishInfo =
	this.tryToFindTopicPublishInfo(msg.getTopic());
	if (topicPublishInfo != null && topicPublishInfo.ok()) {
		Boolean callTimeout = false;
		//3、通过这里可以很明显看出 如果不是同步发送消息 那么消息重试只有1次
		int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 +
		this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
		//4、根据设置的重试次数，循环再去获取服务器主题消息
		for (times = 0; times < timesTotal; times++) {
			MessageQueue mqSelected =
			this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
			beginTimestampPrev = System.currentTimeMillis();
			long costTime = beginTimestampPrev - beginTimestampFirst;
			//5、前后时间对比 如果前后时间差 大于 设置的等待时间 那么直接跳出for循环了 这就
			说明连接超时是不进行多次连接重试的
			if (timeout < costTime) {
				callTimeout = true;
				break;
			}
			//6、如果超时直接报错
			if (callTimeout) {
				throw new RemotingTooMuchRequestException("sendDefaultImpl call
timeout");
			}
		}
	}
```
### 1.4.2 重试总结
通过这段源码很明显可以看出以下几点：
 1. 如果是异步发送那么send次数只有1次
 2. 对于同步而言，超时异常是不会再去重试。
 3. 因为发生重试是在一个for 循环里去重试，所以它是立即重试而不是隔一段时间去重试。
## 1.5 禁止自动创建topic
### 1.5.1 自动创建topic流程
autoCreateTopicEnable 设置为true 标识开启自动创建topic
 1. 消息发送时如果根据topic没有获取到 路由信息，则会根据默认的topic去获取，获取到路由信息后选择一个队列进行发送，发送时报文会带上默认的topic以及默认的队列数量
 2. 消息到达broker后，broker检测没有topic的路由信息，则查找默认topic的路由信息，查到表示开启了自动创建topic，则会根据消息内容中的默认的队列数量在本broker上创建topic，然后进行消息存储
 3. broker创建topic后并不会马上同步给namesrv，而是每30进行汇报一次，更新namesrv上的topic路由信息，producer会每30s进行拉取一次topic的路由信息，更新完成后就可以正常发送消息。更新之前一直都是按照默认的topic查找路由信息
### 1.5.2 为什么不能开启自动创建
上述 broker 中流程会有一个问题，就是在producer更新路由信息之前的这段时间，如果消息只发送到了broker-a，则broker-b上不会创建这个topic的路由信息，broker互相之间不通信。当producer更新之后，获取到的broker列表只有broker-a，就永远不会轮询到broker-b的队列(因为没有路由信息)，所以我们生产通常关闭自动创建broker，而是采用手动创建的方式。
## 1.6 发送端规避
此处发现有可能在实际的生产过程中，RocketMQ 有几台服务器构成的集群。其中有可能是一个主题 TopicA 中的 4 个队列分散在 Broker1、Broker2、Broker3 服务器上。
![请添加图片描述](https://img-blog.csdnimg.cn/6fc42314b426485a8e0031940e9f892c.png)
如果这个时候 Broker2 挂了，我们知道但是生产者不知道（因为生产者客户端每隔 30S 更新一次路由，但是 NamServer 与 Broker 之间的心跳检测间隔是 10S，所以生产者最快也需要 30S 才能感知Broker2 挂了），所以发送到 queue2 的消息会失败，RocketMQ 发现这次消息发送失败后，就会将Broker2排除在消息的选择范围，下次再次发送消息时就不会发送到 Broker2,这样做的目的就是为了提高发送消息的成功率。
# 2 消费端保障
## 2.1 注意幂等性
至少一次送达的消息交付策略，和消息重复消费是一对共生的因果关系。要做到不丢消息就无法避免消息重复消费。原因很简单，试想一下这样的场景：客户端接收到消息并完成了消费，在消费确认过程中发生了通讯错误。从Broker的角度是无法得知客户端是在接收消息过程中出错还是在消费确认过程中出错。为了确保不丢消息，重发消息是唯一的选择。

有了消息幂等消费约定的基础，RocketMQ就能够有针对性地采取一些性能优化措施，例如：并行消费、消费进度同步机制等，这也是RocketMQ性能优异的原因之一。

## 2.2 消息消费模式(从维度划分)
从不同的维度划分，Consumer支持以下消费模式：

 - 广播消费模式下，消息消费失败不会进行重试，消费进度保存在Consumer端
 - 集群消费模式下，消息消费失败有机会进行重试，消费进度集中保存在Broker端
### 2.2.1 集群消费
使用相同 Group ID 的订阅者属于同一个集群，同一个集群下的订阅者消费逻辑必须完全一致（包括 Tag 的使用），这些订阅者在逻辑上可以认为是一个消费节点
![请添加图片描述](https://img-blog.csdnimg.cn/2c07d532c4014b109bb1f39cfab9ce17.png)
注意事项：
 - 消费端集群化部署， 每条消息只需要被处理一次
 - 由于消费进度在服务端维护， 可靠性更高
 - 集群消费模式下，每一条消息都只会被分发到一台机器上处理。如果需要被集群下的每一台机器都处理，请使用广播模式
 - 集群消费模式下，不保证每一次失败重投的消息路由到同一台机器上，因此处理消息时不应该做任何确定性假设
### 2.2.2 广播消费
广播消费指的是：一条消息被多个consumer消费，即使这些consumer属于同一个ConsumerGroup,消息也会被ConsumerGroup中的每个Consumer都消费一次，广播消费中ConsumerGroup概念可以认为在消息划分方面无意义。
![请添加图片描述](https://img-blog.csdnimg.cn/25308c19f1c543f59cca6e072170dc34.png)
注意事项：

 - 广播消费模式下不支持顺序消息
 - 广播消费模式下不支持重置消费位点
 - 每条消息都需要被相同逻辑的多台机器处理
 - 消费进度在客户端维护，出现重复的概率稍大于集群模式
 - 广播模式下，RocketMQ保证每条消息至少被每台客户端消费一次，但是并不会对消费失败的消息进行失败重投，因此业务方需要关注消费失败的情况
 - 广播模式下，客户端每一次重启都会从最新消息消费。客户端在被停止期间发送至服务端的消息将会被自动跳过，请谨慎选择
 - 广播模式下，每条消息都会被大量客户端重复处理，因此推荐尽可能使用集群模式
 - 目前仅java客户端支持广播模式
 - 广播模式下服务端不维护消费进度，因此RocketMQ控制台不支持消息堆积查询、消息堆积报警和订阅关系查询功能
### 2.2.3 集群模式模拟广播
如果业务需要使用广播模式，也可以创建多个 Group ID，用于订阅同一个 Topic。

![请添加图片描述](https://img-blog.csdnimg.cn/d59e6430f6f84be69852d0b2cbee6c11.png)
注意事项：
 - 每条消息都需要被多台机器处理，每台机器逻辑可以相同也可以不一样
 - 消费进度在服务端维护，可靠性高于广播模式
 - 对于一个 Group ID 来说，可以部署一个消费端实例，也可以部署多个消费端实例。当部署多个消费端实例时，实例之间又组成了集群模式（共同分担消费消息）。假设 Group ID 1 部署了三个消费者实例 C1、C2、C3，那么这三个实例将共同分担服务器发送给 Group ID 1 的消息。同时，实例之间订阅关系必须保持一致
## 2.3 消息消费的推、拉模式
RocketMQ消息消费本质上是基于的拉（pull）模式，consumer主动向消息服务器broker拉取消息。
 - 推消息模式下，消费进度的递增是由RocketMQ内部自动维护的
 - 拉消息模式下，消费进度的变更需要上层应用自己负责维护，RocketMQ只提供消费进度保存和查询功能
### 2.3.1 推模式
我们上面使用的消费者都是PUSH模式，也是最常用的消费模式。

由消息中间件（MQ消息服务器代理）主动地将消息推送给消费者；采用Push方式，可以尽可能实时地将消息发送给消费者进行消费。但是，在消费者的处理消息的能力较弱的时候(比如，消费者端的业务系统处理一条消息的流程比较复杂，其中的调用链路比较多导致消费时间比较久。概括起来地说就是“慢消费问题”)，而MQ不断地向消费者Push消息，消费者端的缓冲区可能会溢出，导致异常。

实现方式，代码上使用 DefaultMQPushConsumer

consumer把轮询过程封装了，并注册MessageListener监听器，取到消息后，唤醒MessageListener的consumeMessage()来消费，对用户而言，感觉消息是被推送（push）过来的。主要用的也是这种方式。
### 2.3.2 拉模式
RocketMQ的PUSH模式是由PULL模式来实现的。

由消费者客户端主动向消息中间件（MQ消息服务器代理）拉取消息；采用Pull方式，如何设置Pull消息的频率需要重点去考虑，举个例子来说，可能1分钟内连续来了1000条消息，然后2小时内没有新消息产生（概括起来说就是“消息延迟与忙等待”）。如果每次Pull的时间间隔比较久，会增加消息的延迟，即消息到达消费者的时间加长，MQ中消息的堆积量变大；若每次Pull的时间间隔较短，但是在一段时间内MQ中并没有任何消息可以消费，那么会产生很多无效的Pull请求的RPC开销，影响MQ整体的网络性能。


注意：RocketMQ 4.6.0版本后将弃用DefaultMQPullConsumer，DefaultMQPullConsumer方式需要手动管理偏移量，官方已经被废弃，将在2022年进行删除。现在推荐使用DefaultLitePullConsumer，该类是官方推荐使用的手动拉取的实现类，偏移量提交由RocketMQ管理，不需要手动管理。
## 2.4 消息确认机制
为了保证数据不被丢失，RocketMQ支持消息确认机制，即ack。发送者为了保证消息肯定消费成功，只有使用方明确表示消费成功，RocketMQ才会认为消息消费成功。中途断电，抛出异常等都不会认为成功——即都会重新投递。
### 2.4.1 确认消费
业务实现消费回调的时候，当且仅当此回调函数返回ConsumeConcurrentlyStatus.CONSUME_SUCCESS，RocketMQ才会认为这批消息（默认是1条）是消费完成的。
```java
consumer.registerMessageListener(new MessageListenerConcurrently() {
	@Override
	public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
	ConsumeConcurrentlyContext context) {
		System.out.println(Thread.currentThread().getName() + " Receive New
Messages: " + msgs);
		execute();
		//执行真正消费
		return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
	}
}
)
```
### 2.4.2 消费异常
如果这时候消息消费失败，例如数据库异常，余额不足扣款失败等一切业务认为消息需要重试的场景，只要返回ConsumeConcurrentlyStatus.RECONSUME_LATER ，RocketMQ就会认为这批消息消费失败了。
```java
consumer.registerMessageListener(new MessageListenerConcurrently() {
	@Override
	public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
	ConsumeConcurrentlyContext context) {
		System.out.println(Thread.currentThread().getName() + " Receive New
Messages: " + msgs);
		execute();
		//执行真正消费
		return ConsumeConcurrentlyStatus.RECONSUME_LATER
	}
}
)
```
为了保证消息是肯定被至少消费成功一次，RocketMQ会把这批消息重发回Broker（topic不是原topic而是这个消费组的RETRY topic，可以理解为临时存放点），在延迟的某个时间点（默认是10秒，业务可设置）后，再次投递到这个ConsumerGroup。而如果一直这样重复消费都持续失败到一定次数（默认16次），就会投递到DLQ死信队列。应用可以监控死信队列来做人工干预。
## 2.5 消息重试机制
### 2.5.1 顺序消息的重试
对于顺序消息，当消费者消费消息失败后，消息队列RocketMQ会自动不断地进行消息重试（每次间隔时间为1秒），这时，应用会出现消息消费被阻塞的情况。因此，建议您使用顺序消息时，务必保证应用能够及时监控并处理消费失败的情况，避免阻塞现象的发生。
### 2.5.2 无序消息的重试
无序消息的重试只针对集群消费方式生效；广播方式不提供失败重试特性，即消费失败后，失败消息不再重试，继续消费新的消息。
### 2.5.3 重试次数
消息队列RocketMQ默认允许每条消息最多重试16次，每次重试的间隔时间如下。
| 第几次重试| 与上次重试的间隔时间| 第几次重试| 与上次重试的间隔时间| 
|--|--|--|--|
| 1 | 10秒 | 9 | 7分钟 |
| 2 | 30秒 | 10 | 8分钟 | 
| 3 | 1分钟 | 11 | 9分钟 | 
| 4 | 2分钟 | 12 | 10分钟 | 
| 5 | 3分钟 | 13 | 20分钟 | 
| 6 | 4分钟 | 14 | 30分钟 | 
| 7 | 5分钟 | 15 | 1小时 | 
| 8 | 6分钟 | 16 | 2小时 | 

如果消息重试16次后仍然失败，消息将不再投递。如果严格按照上述重试时间间隔计算，某条消息在一直消费失败的前提下，将会在接下来的4小时46分钟之内进行16次重试，超过这个时间范围消息将不再重试投递。

### 2.5.4 和生产端重试区别
消费者和生产者的重试还是有区别的，主要有两点：

 - 默认重试次数：Product默认是2次，而Consumer默认是16次
 - 重试时间间隔：Product是立刻重试，而Consumer是有一定时间间隔的。它照1S,5S,10S,30S,1M,2M····2H 进行重试

注意：Product在异步情况重试失效，而对于Consumer在广播情况下重试失效。
### 2.5.5 重试配置方式
消费失败后，重试配置方式，集群消费方式下，消息消费失败后期望消息重试，需要在消息监听器接口的实现中明确进行配置（三种方式任选一种）：
方式1：返回RECONSUME_LATER（推荐）
方式2：返回Null
方式3：抛出异常

**无需重试**：集群消费方式下，消息失败后期望消息不重试，需要捕获消费逻辑中可能抛出的异常，最终返回Action.CommitMessage，此后这条消息将不会再重试。
```java
//注册消息监听器
consumer.registerMessageListener(new MessageListenerConcurrently() {
	public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list,
	ConsumeConcurrentlyContext context) {
		//消息处理逻辑抛出异常，消息将重试。
		try {
			doConsumeMessage(list);
		}
		catch (Exception e){
			//捕获消费逻辑中的所有异常，并返回Action.CommitMessage;
			return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
		}
		//业务方正常消费
		return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
	}
}
);
```

