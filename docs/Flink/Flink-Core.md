## 1、DataStream转换
### <1>、映射
采用一个数据元并生成一个数据元，一个map函数，它将输入流的值加倍：
```java
DataStream<Integer> dataStream = //...
dataStream.map(new MapFunction<Integer, Integer>() {
    @Override
    public Integer map(Integer value) throws Exception {
        return 2 * value;
    }
});
```
### <2>、FlatMap
采用一个数据元并生成零个，一个或多个数据元。将句子分割为单词的flatmap函数：

```java
dataStream.flatMap(new FlatMapFunction<String, String>() {
    @Override
    public void flatMap(String value, Collector<String> out)
        throws Exception {
        for(String word: value.split(" ")){
            out.collect(word);
        }
    }
});
```
### <3>、Filter
计算每个数据元的布尔函数，并保存函数返回true的数据元。过滤掉零值的过滤器：

```java
dataStream.filter(new FilterFunction<Integer>() {
    @Override
    public boolean filter(Integer value) throws Exception {
        return value != 0;
    }
});
```
### <4>、KeyBy
逻辑上将流分区为不相交的分区，具有相同Keys的所有记录都分配给同一分区。在内部keyBy()是使用散列分区实现的。指定键有不同的方法。
此转换返回的是KeyedStream，其中包括使用被Keys化妆台所需的KeyedStream

```java
dataStream.keyBy("someKey") // Key by field "someKey"
dataStream.keyBy(0) // Key by the first element of a Tuple
```
<font color=red>注意：如果出现以下情况，则类型不能成为关键：
1、是POJO类型但是不覆盖hashCode()方法并依赖于Object.hashCode()实现
2、是任何类型的数组</font>
### <5>、Reduce
被Keys化数据流上的“滚动”Reduce，将当前数据元与最后一个Reduce的值组合并发出新值。

reduce函数，用于创建部分和的流：

```java
keyedStream.reduce(new ReduceFunction<Integer>() {
    @Override
    public Integer reduce(Integer value1, Integer value2)
    throws Exception {
        return value1 + value2;
    }
});
```
### <6>、折叠
具有初始值的被Keys化数据流上的“滚动“折叠。将当前数据元与最后折叠的值组合并发出新值。

折叠函数，当应用于序列(1,2,3,4,5)时，发出序列”start-1“，”start-2“，”start-1-2-3“，......

```java
DataStream<String> result =
  keyedStream.fold("start", new FoldFunction<Integer, String>() {
    @Override
    public String fold(String current, Integer value) {
        return current + "-" + value;
    }
  });
```
### <7>、聚合
在被Keys化数据流上滚动聚合，min和minBy之间的差异是民返回最小值，而minBy返回该字段中具有最小值的数据元()max和maxBy相同)

```java
keyedStream.sum(0);
keyedStream.sum("key");
keyedStream.min(0);
keyedStream.min("key");
keyedStream.max(0);
keyedStream.max("key");
keyedStream.minBy(0);
keyedStream.minBy("key");
keyedStream.maxBy(0);
keyedStream.maxBy("key");
```
### <8>、Window
可以在已经分区的KeyedStream上定义Windows。Windows根据某些特征(例如在最后5秒内到达的数据)对所有流事件进行分组。有关窗口的完整说明，参见windows。

```java
dataStream.keyBy(0).window(TumblingEventTimeWindows.of(Time.seconds(5))); // Last 5 seconds of data
```
### <9>、WindowAll
Windows可以在常规DataStream上定义，Windows根据某些特征(例如在最后5秒内到达的数据)对所有流事件进行分组。

注意：在许多情况下这是非并行转换。所有记录将收集在windowAll算子的一个任务中。

```java
dataStream.windowAll(TumblingEventTimeWindows.of(Time.seconds(5))); // Last 5 seconds of data
```
### <10>、WindowApply
将一般函数应用于整个窗口，下面是一个手动求和窗口数据元的函数：

注意：如果正在使用windowAll转换，则需要使用AllWindowFunction

```java
windowedStream.apply (new WindowFunction<Tuple2<String,Integer>, Integer, Tuple, Window>() {
    public void apply (Tuple tuple,
            Window window,
            Iterable<Tuple2<String, Integer>> values,
            Collector<Integer> out) throws Exception {
        int sum = 0;
        for (value t: values) {
            sum += t.f1;
        }
        out.collect (new Integer(sum));
    }
});

// applying an AllWindowFunction on non-keyed window stream
allWindowedStream.apply (new AllWindowFunction<Tuple2<String,Integer>, Integer, Window>() {
    public void apply (Window window,
            Iterable<Tuple2<String, Integer>> values,
            Collector<Integer> out) throws Exception {
        int sum = 0;
        for (value t: values) {
            sum += t.f1;
        }
        out.collect (new Integer(sum));
    }
});
```
### <11>、WindowReduce
将函数缩减函数应用于窗口并返回缩小的值。

```java
windowedStream.reduce (new ReduceFunction<Tuple2<String,Integer>>() {
    public Tuple2<String, Integer> reduce(Tuple2<String, Integer> value1, Tuple2<String, Integer> value2) throws Exception {
        return new Tuple2<String,Integer>(value1.f0, value1.f1 + value2.f1);
    }
});
```
### <12>、Window Fold
将函数折叠函数应用于窗口并返回折叠值。示例函数应用于序列(1,2,3,4,5)时将序列折叠为字符串“start-1-2-3-4-5”：

```java
windowedStream.fold("start", new FoldFunction<Integer, String>() {
    public String fold(String current, Integer value) {
        return current + "-" + value;
    }
});
```
### <13>、Windows上的聚合
聚合窗口的内容，min和minBy之间的差异是min返回最小值，minBy返回该字段中具有最小值的数据元

```java
windowedStream.sum(0);
windowedStream.sum("key");
windowedStream.min(0);
windowedStream.min("key");
windowedStream.max(0);
windowedStream.max("key");
windowedStream.minBy(0);
windowedStream.minBy("key");
windowedStream.maxBy(0);
windowedStream.maxBy("key");
```
### <14>、Union
两个或多个数据流的联合，创建包含来自所有流的所有数据元的新流。注意：如果将数据流与自身联合，则会在结果流中获取两次数据元。

```java
dataStream.union(otherStream1, otherStream2, ...);
```
### <15>、Window Join
在给定Keys和公共窗口上连接两个数据流：

```java
dataStream.join(otherStream)
    .where(<key selector>).equalTo(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply (new JoinFunction () {...});
```
### <16>、Interval Join
在给定时间间隔内使用公共Keys关联两个被Key化的数据流的两个数据元e1和e2，以便e1.timestamp + lowerBound <= e2.timestamp <= e1.timestamp + upperBound

```java
// this will join the two streams so that
// key1 == key2 && leftTs - 2 < rightTs < leftTs + 2
keyedStream.intervalJoin(otherKeyedStream)
    .between(Time.milliseconds(-2), Time.milliseconds(2)) // lower and upper bound
    .upperBoundExclusive(true) // optional
    .lowerBoundExclusive(true) // optional
    .process(new IntervalJoinFunction() {...});
```
### <17>、Window CoGroup
DataStream，DataStream→DataStream
在给定Keys和公共窗口上对两个数据流进行CoGroup。

```java
dataStream.coGroup(otherStream)
    .where(0).equalTo(1)
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply (new CoGroupFunction () {...});
```
### <18>、连接
DataStream，DataStream→ConnectedStreams
”连接“两个保存其类型的数据流，连接允许两个流之间的共享状态。

```java
DataStream<Integer> someStream = //...
DataStream<String> otherStream = //...

ConnectedStreams<Integer, String> connectedStreams = someStream.connect(otherStream);
```
### <19>、CoMap、CoFlatMap
类似于数据流上的map和flatmap
```java
connectedStreams.map(new CoMapFunction<Integer, String, Boolean>() {
    @Override
    public Boolean map1(Integer value) {
        return true;
    }

    @Override
    public Boolean map2(String value) {
        return false;
    }
});
connectedStreams.flatMap(new CoFlatMapFunction<Integer, String, String>() {

   @Override
   public void flatMap1(Integer value, Collector<String> out) {
       out.collect(value.toString());
   }

   @Override
   public void flatMap2(String value, Collector<String> out) {
       for (String word: value.split(" ")) {
         out.collect(word);
       }
   }
});
```
### <20>、拆分
根据某些标准将流拆分成两个或更多个流

```java
SplitStream<Integer> split = someDataStream.split(new OutputSelector<Integer>() {
    @Override
    public Iterable<String> select(Integer value) {
        List<String> output = new ArrayList<String>();
        if (value % 2 == 0) {
            output.add("even");
        }
        else {
            output.add("odd");
        }
        return output;
    }
});
```
### <21>、选择
SplitStream→DataStream
从拆分中选择一个或多个流。

```java
SplitStream<Integer> split;
DataStream<Integer> even = split.select("even");
DataStream<Integer> odd = split.select("odd");
DataStream<Integer> all = split.select("even","odd");
```
### <22>、迭代
DataStream→IterativeStream→DataStream

通过将一个算子的输出重定向到某个先前的算子，在流中创建”反馈“循环。对于定义不断更新模型的算法特别有用。以下代码以流开头并连续应用迭代体。大于0的数据元将被发送回反馈通道，其余数据元将向下游转发。

```java
IterativeStream<Long> iteration = initialStream.iterate();
DataStream<Long> iterationBody = iteration.map (/*do something*/);
DataStream<Long> feedback = iterationBody.filter(new FilterFunction<Long>(){
    @Override
    public boolean filter(Integer value) throws Exception {
        return value > 0;
    }
});
iteration.closeWith(feedback);
DataStream<Long> output = iterationBody.filter(new FilterFunction<Long>(){
    @Override
    public boolean filter(Integer value) throws Exception {
        return value <= 0;
    }
});
```
### <23>、提取时间戳
从记录中提取时间戳，以便使用事件时间语义的窗口。

```java
stream.assignTimestamps (new TimeStampExtractor() {...});
```
### <24>、Project
从元组中选择字段的子集：

```java
DataStream<Tuple3<Integer, Double, String>> in = // [...]
DataStream<Tuple2<String, Integer>> out = in.project(2,0);
```
## 2、物理分区
### <1>、自定义分区
使用用户定义的分区程序为每个数据元选择目标任务

```java
dataStream.partitionCustom(partitioner, "someKey");
dataStream.partitionCustom(partitioner, 0);
```
### <2>、随机分区
根据均匀分布随机分配 数据元

```java
dataStream.shuffle();
```
### <3>、Rebalance(循坏分区)
分区数据元循环，每个分区创建相等的负载。在存在数据偏斜时用于性能优化。

```java
dataStream.rebalance();
```
### <4>、重新调整
分区数据元，循环，到下游算子操作的子集。如果您希望拥有管道，例如从源的每个并行实例扇出到多个映射器的子集以分配负载但又不希望发生rebalance()会产生完全Rebalance ，那么这非常有用。这将仅需要本地数据传输而不是通过网络传输数据，具体取决于其他配置值，例如TaskManagers的插槽数。

上游算子操作发送数据元的下游算子操作的子集取决于上游和下游算子操作的并行度。例如，如果上游算子操作具有并行性2并且下游算子操作具有并行性6，则一个上游算子操作将分配元件到三个下游算子操作，而另一个上游算子操作将分配到其他三个下游算子操作。另一方面，如果下游算子操作具有并行性2而上游 算子操作具有并行性6，则三个上游算子操作将分配到一个下游算子操作，而其他三个上游算子操作将分配到另一个下游算子操作。

在不同并行度不是彼此的倍数的情况下，一个或多个下游 算子操作将具有来自上游 算子操作的不同数量的输入。

参阅此图获取上例中国连接模式的可视化：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191006133952965.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDI0MDM3MA==,size_16,color_FFFFFF,t_70)

```java
dataStream.rescale();
```
### <5>、广播
向每个分区广播数据元

```java
dataStream.broadcast();
```
## 3、数据集转换
数据转换将一个或多个DataSet转换为新的DataSet。程序将多个转换组合到复杂的程序中。
### <1>、Map
采用一个数据元并生成一个数据元。

```java
data.map(new MapFunction<String, Integer>() {
  public Integer map(String value) { return Integer.parseInt(value); }
});
```

转换：采用一个元素并生成一个元素

```java
data.map { x => x.toInt }
```
### <2>、FlatMap
采用一个数据元并生成零个，一个或多个数据元。
```java
data.flatMap(new FlatMapFunction<String, String>() {
  public void flatMap(String value, Collector<String> out) {
    for (String s : value.split(" ")) {
      out.collect(s);
    }
  }
});
```

转换：采用一个元素并生成零个、一个或多个元素

```java
data.flatMap { str => str.split(" ") }
```
### <3>、MapPartition
在单个函数调用中转换并分区，该函数将分区作为Iterable流来获取，并且可以生成任意数量的结果值。每个分区中的数据元数量取决于并行度和先前的算子操作。

```java
data.mapPartition(new MapPartitionFunction<String, Long>() {
  public void mapPartition(Iterable<String> values, Collector<Long> out) {
    long c = 0;
    for (String s : values) {
      c++;
    }
    out.collect(c);
  }
});
```

转换：在单个函数调用中转换并分区，该函数将分区作为迭代器，并可以生成任意数量的结果值，每个分区中的元素数量取决于并行度和先前的算子操作。

```java
data.mapPartition { in => in map { (_, 1) } }
```
### <4>、Filter
计算每个数据的布尔函数，并保存函数返回true的数据元

**注意：系统假定该函数不会修改应用谓词的数据元，违反此假设可能会导致错误的结果。**
```java
data.filter(new FilterFunction<Integer>() {
  public boolean filter(Integer value) { return value > 1000; }
});
```

转换：计算每个元素的布尔函数，并保存函数返回true的元素。

```java
data.filter { _ > 1000 }
```
### <5>、Reduce
通过将两个数据元重复组合成一个数据元，将一组数据元组合成一个数据元。Reduce可以应用于完整数据集或分组数据集。

```java
data.reduce(new ReduceFunction<Integer> {
  public Integer reduce(Integer a, Integer b) { return a + b; }
});
```
如果将reduce应用于分组数据集，则可以通过提供CombineHintto来指定运行时执行reduce的组合阶段的方式setCombineHint。大多数情况下，基于散列的策略应该更快，特别是如果不同键的数量与输入数据元的数量相比较小。

转换：通过将两个元素重复组合成一个元素，将一组元素组合成一个元素。Reduce可以应用于完整数据集或分组数据集。

```java
data.reduce { _ + _ }
```
### <6>、ReduceGroup
将一组数据元组合成一个或多个数据元。ReduceGroup可以应用于完整数据集或分组数据集。

```java
data.reduceGroup(new GroupReduceFunction<Integer, Integer> {
  public void reduce(Iterable<Integer> values, Collector<Integer> out) {
    int prefixSum = 0;
    for (Integer i : values) {
      prefixSum += i;
      out.collect(prefixSum);
    }
  }
});
```

转换：将一组元素组合成一个或多个元素。ReduceGroup可以用于完整数据集或分组数据集。

```java
data.reduceGroup { elements => elements.sum }
```
### <7>、Aggregate
将一组聚合为单个值。聚合函数可以被认为是内置的reduce函数，聚合可以应用于完整数据集或分组数据集。

```java
Dataset<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input.aggregate(SUM, 0).and(MIN, 2);
```
可以使用简写语法进行最小最大和总和聚合。

```java
Dataset<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input.sum(0).andMin(2);
```
### <8>、Distinct
返回数据集的不同数据元，相对于数据元的所有字段或字段子集从输入DataSet中删除重复条目。

```java
data.distinct();
```
使用reduce函数实现Distinct。可以通过CombineHintto来指定运行时执行reduce的组合阶段的方式setCombineHint。大多数情况下，基于散列的策略应该更快，特别是如果不同键的数量与输入数据元的数量相比较小。
### <9>、Join
通过创建在其键上相等的所有数据元对来连接两个数据集。可选的使用JoinFunction将数据元对转换为单个数据元或使用FlatJoinFunction将数据元对转换为任意多个数据元。

```java
result = input1.join(input2)	
               .where(0)       // key of the first input (tuple field 0)
               .equalTo(1);    // key of the second input (tuple field 1)
```

### <10>、Outer Join
在两个数据集上执行左右或全外连接。外连接类似常规连接并创建在其键上相等的所有数据元对。此外如果在另一侧没有找到匹配的Keys，则保存外部侧的记录。匹配数据元对被赋予JoinFunction以将数据元对转换为单个数据元，或者转换为FlatJoinFunction以将数据元对转换为任意多个数据元。

```java
input1.leftOuterJoin(input2) // rightOuterJoin or fullOuterJoin for right or full outer joins
      .where(0)              // key of the first input (tuple field 0)
      .equalTo(1)            // key of the second input (tuple field 1)
      .with(new JoinFunction<String, String, String>() {
          public String join(String v1, String v2) {
             // NOTE:
             // - v2 might be null for leftOuterJoin
             // - v1 might be null for rightOuterJoin
             // - v1 OR v2 might be null for fullOuterJoin
          }
      });
```

```java
val joined = left.leftOuterJoin(right).where(0).equalTo(1) {
   (left, right) =>
     val a = if (left == null) "none" else left._1
     (a, right)
  }
```
### <11>、CoGroup
reduce算子操作的二维变体，将一个或多个字段上的每个输入分组然后关联组。每对组调用转换函数。

```java
data1.coGroup(data2)
     .where(0)
     .equalTo(1)
     .with(new CoGroupFunction<String, String, String>() {
         public void coGroup(Iterable<String> in1, Iterable<String> in2, Collector<String> out) {
           out.collect(...);
         }
      });
```

```java
data1.coGroup(data2).where(0).equalTo(1)
```
### <12>、Cross
构建两个输入的笛卡尔积，创建所有数据元对，可以选择使用CrossFunction将数据元对转换为单个数据元。

```java
DataSet<Integer> data1 = // [...]
DataSet<String> data2 = // [...]
DataSet<Tuple2<Integer, String>> result = data1.cross(data2);
```
<font color=red>交叉是一个潜在的非常计算密集型算子操作它甚至可以挑战大的计算集群。建议使用crossWithTiny（）和crossWithHuge（）来提示系统的DataSet大小。</font>
### <13>、Union
生成两个数据集的并集。

```java
DataSet<String> data1 = // [...]
DataSet<String> data2 = // [...]
DataSet<String> result = data1.union(data2);
```
### <14>、Rebalance
均匀的Rebalance数据集的并行分区以消除数据偏差。只有类似Map的转换可能会遵循Rebalance转换。

```java
DataSet<String> in = // [...]
DataSet<String> result = in.rebalance()
                           .map(new Mapper());
```
### <15>、Hash-Partition
散列分区给定键上的数据集。键可以指定为位置键，表达键和键选择器函数。

```java
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.partitionByHash(0)
                            .mapPartition(new PartitionMapper());
```
### <16>、Range-Partition
Range-Partition给定键上的数据集，键可以指定为位置键，表达键和键选择器函数。

```java
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.partitionByRange(0)
                            .mapPartition(new PartitionMapper());
```
### <17>、Custom Partitioning
手动指定分区。
注意：此方法仅适用于单个字段建。

```java
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.partitionCustom(Partitioner<K> partitioner, key)
```
### <18>、Sort Partition
本地按指定顺序对指定字段上的数据集的所有分区进行排序，可以将字段指定为元组位置活泼字段表达式。通过链接sortPartition()调用来完成对多个字段的排序。

```java
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.sortPartition(1, Order.ASCENDING)
                            .mapPartition(new PartitionMapper());
```
### <19>、First-n
返回数据集的前n个数据元，First-n可以应用于常规数据集，分组数据集或分组排序数据集。分组键可以指定为键选择器函数或字段位置键。

```java
DataSet<Tuple2<String,Integer>> in = // [...]
// regular data set
DataSet<Tuple2<String,Integer>> result1 = in.first(3);
// grouped data set
DataSet<Tuple2<String,Integer>> result2 = in.groupBy(0)
                                            .first(3);
// grouped-sorted data set
DataSet<Tuple2<String,Integer>> result3 = in.groupBy(0)
                                            .sortGroup(1, Order.ASCENDING)
                                            .first(3);

```
### <20>、MinBy / MaxBy
从一组数据中选择一个元组，其元组的一个或多个字段的值最小。用于比较的字段必须是有效的关键字段，即可比较。如果多个元组具有最小字段值，则返回这些元组的任意元组。
MinBy / MaxBy可以应用于完整数据集火分组数据集。

```java
DataSet<Tuple3<Integer, Double, String>> in = // [...]
// a DataSet with a single tuple with minimum values for the Integer and String fields.
DataSet<Tuple3<Integer, Double, String>> out = in.minBy(0, 2);
// a DataSet with one tuple for each group with the minimum value for the Double field.
DataSet<Tuple3<Integer, Double, String>> out2 = in.groupBy(2)
                                                  .minBy(1);
```
