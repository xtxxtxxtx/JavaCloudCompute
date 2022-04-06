## 五、Time和Window
### 5.1、Time
在Flink流式处理中会设计到时间的不同概念如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191018185515418.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDI0MDM3MA==,size_16,color_FFFFFF,t_70)
**Event Time**：是事件创建的时间。通常由事件中的时间戳描述，例如采集的日志数据，每一条日志都会记录自己生成时间，Flink通过时间戳分配器访问事件时间戳。
**Ingestion Time**：数据进入Flink时间。
**Processing Time**：每一个执行基于时间操作的算子的本地系统时间，与机器相关默认时间属性是Processing Time。

对于业务来说要统计1min内故障日志个数eventTime时间是最有意义的因为我们要根据日志生成时间进行统计。
### 5.2、Window
#### 1、概述
Streaming流式计算是一种被设计用于处理无限数据集的数据处理引擎而无限数据集是指一种不断增长的本质上无限的数据集，而Window是一种切割无限数据为有限块进行处理的手段。

Window是无限数据流处理的核心，Window将一个无限的stream拆分成有限大小的buckets桶，可以在这些桶上做计算操作。
#### 2、Window类型
Window分为两类：

 - CountWindow：按照指定的数据条数生成一个Window与时间无关。
 - TimeWindow：按照时间生成Window

对于TimeWindow可以根据窗口实现原理的不同分成三类：滚动窗口(Tumbling Window)、滑动窗口(Sliding Window)和会话窗口(Session Window)。

 1. 滚动窗口
 
 将数据依据固定的窗口长度对数据进行切片。
 **特点：时间对齐、 窗口长度固定、没有重叠。**
 滚动窗口分配器将每个元素分配到一个指定窗口大小的窗口中滚动窗口有一个固定的大小并且不会出现重叠。
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20191018191136144.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDI0MDM3MA==,size_16,color_FFFFFF,t_70)
 适用场景：适合做BI统计等(每个时间段的聚合计算)。
 
 2. 滑动窗口
 
 滑动窗口是固定窗口的更定义的一种形式滑动窗口由固定的窗口长度和滑动间隔组成。
 特点：时间对齐、窗口长度固定，有重叠。
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20191018191354107.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDI0MDM3MA==,size_16,color_FFFFFF,t_70)
 适用场景：对最近一个时间段内的统计
 
 3. 会话窗口

由一系列事件组合一个指定时间长度的 timeout 间隙组成，类似于 web 应用的session，也就是一段时间没有接收到新数据就会生成新的窗口。
特点： 时间无对齐
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191018191730170.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDI0MDM3MA==,size_16,color_FFFFFF,t_70)
### 5.3、Window API
#### 1、CountWindow
CountWindow 根据窗口中相同 key 元素的数量来触发执行，执行时只计算元素数量达到窗口大小的 key 对应的结果。

<font color=red>注意： CountWindow 的 window_size 指的是相同 Key 的元素的个数，不是输入的所有元素的总数。</font>

1、滚动窗口
默认的CountWindow是一个滚动窗口，只需要指定窗口大小即可，当元素数量达到窗口大小时，就会触发窗口的执行。

```java
// 获取执行环境
val env = StreamExecutionEnvironment.getExecutionEnvironment
// 创建 SocketSource
val stream = env.socketTextStream("localhost", 11111)
// 对 stream 进行处理并按 key 聚合
val streamKeyBy = stream.map(item => (item.split(" ")(0), item.split(" ")(1).toLong)).keyBy(0)

// 引入滚动窗口
// 这里的 5 指的是 5 个相同 key 的元素计算一次
val streamWindow = streamKeyBy.countWindow(5)
// 执行聚合操作
val streamReduce = streamWindow.reduce(
(item1, item2) => (item1._1, item1._2 + item2._2)
)

streamReduce.print()
// 执行程序
env.execute("TumblingWindow")
```
2、滑动窗口
滑动窗口和滚动窗口的函数名是完全一致的，只是在传参数时需要传入两个参数，一个是 window_size，一个是 sliding_size。

```java
// 获取执行环境
val env = StreamExecutionEnvironment.getExecutionEnvironment

// 创建 SocketSource
val stream = env.socketTextStream("localhost", 11111)

// 对 stream 进行处理并按 key 聚合
val streamKeyBy = stream.map(item => (item.split(" ")(0), item.split(" ")(1).toLong)).keyBy(0)

// 引入滚动窗口
// 当相同 key 的元素个数达到 2 个时，触发窗口计算，计算的窗口范围为 5
val streamWindow = streamKeyBy.countWindow(5,2)

// 执行聚合操作
val streamReduce = streamWindow.reduce(
(item1, item2) => (item1._1, item1._2 + item2._2)
)

streamReduce.print()

env.execute("TumblingWindow")
}
```
#### 2、TimeWindow
TimeWindow是将指定时间范围内的所有数据组成一个window，一次对一个window里面的所有数据进行计算。
1、滚动窗口
Flink 默认的时间窗口根据 Processing Time 进行窗口的划分，将Flink获取到的数据根据进入 Flink 的时间划分到不同的窗口中。
```java
// 获取执行环境
val env = StreamExecutionEnvironment.getExecutionEnvironment
// 创建 SocketSource
val stream = env.socketTextStream("localhost", 11111)
// 对 stream 进行处理并按 key 聚合
val streamKeyBy = stream.map(item => (item, 1)).keyBy(0)
// 引入时间窗口
val streamWindow = streamKeyBy.timeWindow(Time.seconds(5))
// 执行聚合操作
val streamReduce = streamWindow.reduce(
(item1, item2) => (item1._1, item1._2 + item2._2)
)
// 将聚合数据写入文件
streamReduce.print()
// 执行程序
env.execute("TumblingWindow")
```
2、滑动窗口
滑动窗口和滚动窗口的函数名是完全一致的，只是在传参数时需要传入两个参数，一个是 window_size，一个是 sliding_size。
```java
// 获取执行环境
val env = StreamExecutionEnvironment.getExecutionEnvironment
// 创建 SocketSource
val stream = env.socketTextStream("localhost", 11111)
// 对 stream 进行处理并按 key 聚合
val streamKeyBy = stream.map(item => (item, 1)).keyBy(0)
// 引入滚动窗口
val streamWindow = streamKeyBy.timeWindow(Time.seconds(5), Time.seconds(2))
// 执行聚合操作
val streamReduce = streamWindow.reduce(
	(item1, item2) => (item1._1, item1._2 + item2._2)
)
// 将聚合数据写入文件
streamReduce.print()
// 执行程序
env.execute("TumblingWindow")
```
#### 3、Window Reduce
WindowedStream → DataStream：给 window 赋一个reduce功能的函数，并返回一个聚合的结果。
```java
// 获取执行环境
val env = StreamExecutionEnvironment.getExecutionEnvironment
// 创建 SocketSource
val stream = env.socketTextStream("localhost", 11111)
// 对 stream 进行处理并按 key 聚合
val streamKeyBy = stream.map(item => (item, 1)).keyBy(0)
// 引入时间窗口
val streamWindow = streamKeyBy.timeWindow(Time.seconds(5))
// 执行聚合操作
val streamReduce = streamWindow.reduce(
	(item1, item2) => (item1._1, item1._2 + item2._2)
)
// 将聚合数据写入文件
streamReduce.print()
// 执行程序
env.execute("TumblingWindow")
```
#### 4、Window Fold
WindowedStream → DataStream：给窗口赋一个fold功能的函数，并返回一个fold后的结果。

```java
// 获取执行环境
val env = StreamExecutionEnvironment.getExecutionEnvironment
// 创建 SocketSource
val stream = env.socketTextStream("localhost", 11111,'\n',3)
// 对 stream 进行处理并按 key 聚合
val streamKeyBy = stream.map(item => (item, 1)).keyBy(0)
// 引入滚动窗口
val streamWindow = streamKeyBy.timeWindow(Time.seconds(5))
// 执行 fold 操作
val streamFold = streamWindow.fold(100){
(begin, item) =>
		begin + item._2
}
// 将聚合数据写入文件
streamFold.print()

// 执行程序
env.execute("TumblingWindow")
```
#### 5、Aggregation on Window
WindowedStream → DataStream：对一个 window 内的所有元素做聚合操作。
min 和 minBy 的区别是 min 返回的是最小值，而 minBy 返回的是包含最小值字段的元素(同样的原理适用于 max 和 maxBy)。

```java
// 获取执行环境
val env = StreamExecutionEnvironment.getExecutionEnvironment
// 创建 SocketSource
val stream = env.socketTextStream("localhost", 11111)
// 对 stream 进行处理并按 key 聚合
val streamKeyBy = stream.map(item => (item.split(" ")(0), item.split(" ")(1))).keyBy(0)
// 引入滚动窗口
val streamWindow = streamKeyBy.timeWindow(Time.seconds(5))
// 执行聚合操作
val streamMax = streamWindow.max(1)
// 将聚合数据写入文件
streamMax.print()
// 执行程序
env.execute("TumblingWindow")
```
## 六、EventTime与Window
### 6.1、EventTime的引入
在 Flink 的流式处理中， 绝大部分的业务都会使用 eventTime，一般只在eventTime 无法使用时，才会被迫使用 ProcessingTime 或者 IngestionTime。

```java
val env = StreamExecutionEnvironment.getExecutionEnvironment

// 从调用时刻开始给 env 创建的每一个 stream 追加时间特征
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
```
### 6.2、Watermark
#### 1、概念
Watermark 是一种衡量 Event Time 进展的机制，它是数据本身的一个隐藏属性，数据本身携带着对应的 Watermark。

Watermark 是用于处理乱序事件的 ， 而正确的处理乱序 事 件 ， 通常用Watermark 机制结合 window 来实现。

数据流中的 Watermark 用于表示 timestamp小于 Watermark 的数据，都已经到达了，因此， window 的执行也是由Watermark触发的。

Watermark可以理解成一个延迟触发机制，我们可以设置 Watermark 的延时时长 t，每次系统会校验已经到达的数据中最大的maxEventTime，然后认定eventTime小于maxEventTime - t 的所有数据都已经到达，如果有窗口的停止时间等于 maxEventTime – t，那么这个窗口被触发执行

有序流的 Watermarker 如下图所示：（ Watermark 设置为 0）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191018200025832.png)
乱序流的 Watermarker 如下图所示：（ Watermark 设置为 2）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191018200050650.png)
当 Flink 接收到每一条数据时，都会产生一条 Watermark，这条 Watermark就等于当前所有到达数据中的maxEventTime - 延迟时长， 也就是说Watermark是由数据携带的，一旦数据携带的Watermark 比当前未触发的窗口的停止时间要晚，那么就会触发相应窗口的执行。由于Watermark 是由数据携带的，因此如果运行过程中无法获取新的数据，那么没有被触发的窗口将永远都不被触发。
#### 2、Watermark引入

```java
val env = StreamExecutionEnvironment.getExecutionEnvironment
// 从调用时刻开始给 env 创建的每一个 stream 追加时间特征
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

val stream = env.socketTextStream("localhost",11111).assignTimestampsAndWatermarks(
	new BoundedOutOfOrdernessTimestampExtractor[String(Time.milliseconds(200)) {
	override def extractTimestamp(t: String): Long = {
		// EventTime 是日志生成时间，我们从日志中解析 EventTime
		t.split(" ")(0).toLong
	}
})
```
### 6.3、EventTime Window API
Window 的设定无关数据本身，而是系统定义好了的，也就是说， Window 会一直按照指定的时间间隔进行划分，不论这个 Window 中有没有数据， EventTime 在这个 Window 期间的数据会进入这个 Window。

Window在以下条件满足的时候被触发执行：

 - watermark 时间 >= window_end_time；
 - 在[window_start_time,window_end_time)中有数据存在。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191018202530560.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDI0MDM3MA==,size_16,color_FFFFFF,t_70)
#### 1、滚动窗口
```java
// 获取执行环境
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

// 创建 SocketSource
val stream = env.socketTextStream("localhost", 11111)
// 对 stream 进行处理并按 key 聚合
val streamKeyBy = stream.assignTimestampsAndWatermarks(
	new BoundedOutOfOrdernessTimestampExtractor[String(Time.milliseconds(3000)) {
	override def extractTimestamp(element: String): Long = {
		val sysTime = element.split(" ")(0).toLong
		println(sysTime)
		sysTime
		}}).map(item => (item.split(" ")(1), 1)).keyBy(0)
		
// 引入滚动窗口
val streamWindow = streamKeyBy.window(TumblingEventTimeWindows.of(Time.seconds(10)))

// 执行聚合操作
val streamReduce = streamWindow.reduce(
	(item1, item2) => (item1._1, item1._2 + item2._2)
)

// 将聚合数据写入文件
streamReduce.print

// 执行程序
env.execute("TumblingWindow")
```
结果是按照EventTime的时间窗口计算得出的，而无关系统的时间（包括输入的快慢） 。
#### 2、滑动窗口
```java
// 获取执行环境
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

// 创建 SocketSource
val stream = env.socketTextStream("localhost", 11111)
// 对 stream 进行处理并按 key 聚合
val streamKeyBy = stream.assignTimestampsAndWatermarks(
new BoundedOutOfOrdernessTimestampExtractor[String(Time.milliseconds(0)) {
	override def extractTimestamp(element: String): Long = {
		val sysTime = element.split(" ")(0).toLong
		println(sysTime)
		sysTime
		}}).map(item => (item.split(" ")(1), 1)).keyBy(0)

// 引入滚动窗口
val streamWindow = streamKeyBy.window(SlidingEventTimeWindows.of(Time.seconds(10),
Time.seconds(5)))

// 执行聚合操作
val streamReduce = streamWindow.reduce(
	(item1, item2) => (item1._1, item1._2 + item2._2)
)

streamReduce.print

env.execute("TumblingWindow")
```
#### 3、会话窗口
相邻两次数据的EventTime的时间差超过指定的时间间隔就会触发执行。
```java
// 获取执行环境
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

// 创建 SocketSource
val stream = env.socketTextStream("localhost", 11111)

// 对 stream 进行处理并按 key 聚合
val streamKeyBy = stream.assignTimestampsAndWatermarks(
	new BoundedOutOfOrdernessTimestampExtractor[String(Time.milliseconds(0)) {
		override def extractTimestamp(element: String): Long = {
			val sysTime = element.split(" ")(0).toLong
			println(sysTime)
			sysTime
			}}).map(item => (item.split(" ")(1), 1)).keyBy(0)

// 引入滚动窗口
val streamWindow = streamKeyBy.window(EventTimeSessionWindows.withGap(Time.seconds(5)))

// 执行聚合操作
val streamReduce = streamWindow.reduce(
	(item1, item2) => (item1._1, item1._2 + item2._2)
)

streamReduce.print

env.execute("TumblingWindow")
```
