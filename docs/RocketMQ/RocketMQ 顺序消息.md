# 顺序类型
## 无序消息
无序消息也指普通的消息，Producer 只管发送消息，Consumer 只管接收消息，至于消息和消息之间的顺序并没有保证。

 - Producer 依次发送 orderId 为1、2、3的消息
 - Consumer 接到的消息顺序有可能是1、2、3，也有可能是2、1、3等情况，这就是普通消息。
## 全局顺序
对于指定的一个 Topic，所有消息按照严格的先入先出的顺序进行发布和消费。
比如 Producer 发送 orderId 1,3,2的消息，那么 Consumer 也必须要按照1,3,2的顺序进行消费。
![请添加图片描述](https://img-blog.csdnimg.cn/a679264f335844fcab97ac97ae2edd41.png)
## 局部顺序
在实际开发场景中，并不需要消息完全按照完全按的先进先出，而是某些消息保证先进先出就可以了。
就好比一个订单涉及 订单生成、订单支付、订单完成。不管其他订单，只保证同样订单ID能保证这个顺序就可以。
![请添加图片描述](https://img-blog.csdnimg.cn/a5ee0c8b38c94336a9aa3ff167b229be.png)
# RocketMQ 顺序消息
RocketMQ 可以严格的保证消息顺序。但这个顺序不是全局顺序，只是分区顺序。要实现全局顺序只能有一个分区。因此发送消息时候，消息发送默认是会采用轮询的方式发送到不通的分区。
## 实现原理
生产的Message最终会存放在Queue中，如果一个Topic关联了四个queue，如果我们不指定消息往哪个队列里放，那么默认是平均分配消息到四个queue。
假如存在十条消息，那么这十条消息会平均分配在这四个queue上，那么每个queue大概放2个左右。重要的一点是：同一个queue存储在里面的message是按照先进先出的原则。
![请添加图片描述](https://img-blog.csdnimg.cn/bb2308f701db46c7ad9758ccd128befd.png)
不同地区使用不同的queue，只要保证同一个地区的订单放在同一个queue，这样就可以保证消费者先进先出。
![请添加图片描述](https://img-blog.csdnimg.cn/176b465b76954fcca64d25eb3c170d5b.png)
此时可以保证局部顺序，即同一订单按照先后顺序放到同一个queue，那么获取消息时候就可保证先进先出。
## 如何保证集群有序
最关键的一点是在一个消费者集群情况下，消费者1先去queue获取消息，获取到北京订单1，然后消费者2去queue获取到的是北京订单2。

获取信息顺序可以保证，但关键是先拿到不代表先消费完。会存在虽然消费者1先获取到北京订单1但由于网络原因，消费者2比真正的先消费消息。

解决该问题使用到的是分布式锁：RocketMQ 采用的是分段锁，不是锁整个Broker而是锁里面的单个queue，因为只要锁单个queue就可保证局部顺序消费。

因此最终的消费者逻辑是：

 - 消费者1去queue获取订单生成，就锁住了整个queue，只有它消费完成并返回成功后，锁才会释放。
 - 然后下一个消费者去获取到订单支付，同样锁住当前queue，这样的一个过程真正保证对同一个queue能够真正意义上的顺序消费，而不仅仅是顺序取出。
## 消息类型对比
|topic消息类型|支持事务消息|支持定时延时消息|性能|
|--|--|--|--|
| 无序消息(普通、事务、定时/延时) | 是 |是 |最高|
| 分区顺序消息 | 否 |否|高|
| 全局顺序消息| 否 |否|一般|

发送方式对比
|topic消息类型|支持可靠同步发送|支持可靠异步发送|支持oneway发送|
|--|--|--|--|
| 无序消息(普通、事务、定时/延时) | 是 |是 |是|
| 分区顺序消息 | 是 |否|否|
| 全局顺序消息| 是 |否|否|
## 注意事项

 1. 顺序消息暂不支持广播模式
 2. 顺序消息不支持异步发送方式，否则将无法严格保证顺序
 3. 建议同一个Group ID 只对应一种类型的 Topic，即不同时用于顺序消息和无序消息的收发
 4. 对于全局顺序消息，建议创建 broker个数大于等于2

## 代码示例
主要实现两点：

 1. 生产者同一订单id的订单放到同一个queue
 2. 消费者同一个queue取出消息时候锁住整个queue，直到消费后再解锁

实体类：

```java
public class Order {
    
    private String orderId;
    private String orderName;
    
    public Order(String orderId, String orderName) {
        this.orderId = orderId;
        this.orderName = orderName;
    }

    public String getOrderId() {
        return orderId;
    }

    public void setOrderId(String orderId) {
        this.orderId = orderId;
    }

    public String getOrderName() {
        return orderName;
    }

    public void setOrderName(String orderName) {
        this.orderName = orderName;
    }
}
```
Producer

```java
public class OrderProducer {
    private static final List<Order> list = new ArrayList<>();

    static {
        list.add(new Order("XXX001", "订单创建"));
        list.add(new Order("XXX001", "订单付款"));
        list.add(new Order("XXX001", "订单完成"));
        list.add(new Order("XXX002", "订单创建"));
        list.add(new Order("XXX002", "订单付款"));
        list.add(new Order("XXX002", "订单完成"));
        list.add(new Order("XXX003", "订单创建"));
        list.add(new Order("XXX003", "订单付款"));
        list.add(new Order("XXX003", "订单完成"));
    }

    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("rocket_test_consumer_group");
        producer.setNamesrvAddr("127.0.0.1;9876");
        producer.start();

        for (int i = 0; i < list.size(); i++) {
            Order order = list.get(i);
            Message message = new Message("topicTest", order.getOrderId(), order.toString().getBytes(RemotingHelper.DEFAULT_CHARSET));
            SendResult result = producer.send(message, new MessageQueueSelector() {
                @Override
                /**
                 * hash队列选择，简化来说就是用index%一个 固定的值，使得同一tag的消息能发送到同一个queue
                 */
                public MessageQueue select(List<MessageQueue> list, Message message, Object o) {
                    String orderId = (String) o;
                    int hashCode = orderId.hashCode();
                    hashCode = Math.abs(hashCode);
                    long index = hashCode % list.size();
                    return list.get((int) index);
                }
            }, order.getOrderId());
            System.out.println("发送状态:" + result.getSendStatus() + ",存储queue:" + result.getMessageQueue().getQueueId()
                    + ",orderId:" + order.getOrderId() + ",name:" + order.getOrderName());
        }
        producer.shutdown();
    }
}
```
Consumer
上面说过，消费者真正要达到消费顺序需要使用分布式锁，因此这里需要使用MessageListenerOrderly替换之前的MessageListenerConcurrently，因为其内部实现了分布式锁。
```java
public class OrderConsumer {

    private static final Random random = new Random();

    public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("rocket_test_consumer_group");
        consumer.setNamesrvAddr("127.0.0.1:9876");
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        consumer.subscribe("topicTest", "*");

        consumer.registerMessageListener(new MessageListenerOrderly() {
            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> list, ConsumeOrderlyContext consumeOrderlyContext) {
                if (list != null) {
                    for (MessageExt messageExt : list) {
                        try {
                            try {
                                TimeUnit.SECONDS.sleep(random.nextInt(10));
                            } catch (Exception e) {
                                e.printStackTrace();
                            }
                            // 获取到接收的消息
                            String message = new String(messageExt.getBody(), RemotingHelper.DEFAULT_CHARSET);

                            // 获取队列id
                            int queueId = consumeOrderlyContext.getMessageQueue().getQueueId();

                            System.out.println("consumer-线程名称:" + Thread.currentThread().getId()
                                    + ",接收queueId:" + queueId + ",消息:" + message);
                        } catch (UnsupportedEncodingException e) {
                            e.printStackTrace();
                        }
                    }
                }
                return ConsumeOrderlyStatus.SUCCESS;
            }
        });
        consumer.start();
        System.out.println("消费者已经启动");
    }
}
```

