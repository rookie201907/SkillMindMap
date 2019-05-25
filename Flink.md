## Flink脑图

#### 数据流API

如下是一个通过Scala版本的数据流API编写的Flink程序代码 该代码实现每5秒计算传感器的平均温度

```scala
case class SensorReading(id: String, timestamp: Long, temperature: Double)
/** Object that defines the DataStream program in the main() method */
object AverageSensorReadings {
  /** main() defines and executes the DataStream program */
  def main(args: Array[String]) {
    // set up the streaming execution environment
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    // use event time for the application
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
    // configure watermark interval
    env.getConfig.setAutoWatermarkInterval(1000L)
    // ingest sensor stream
    val sensorData: DataStream[SensorReading] = env
      // SensorSource generates random temperature readings
      .addSource(new SensorSource)
      // assign timestamps and watermarks which are required for event time
      .assignTimestampsAndWatermarks(new SensorTimeAssigner)
    val avgTemp: DataStream[SensorReading] = sensorData
      // convert Fahrenheit to Celsius using an inlined map function
      .map( r ⇒
      SensorReading(r.id, r.timestamp, (r.temperature - 32) * (5.0 / 9.0)) )
      // organize stream by sensorId
      .keyBy(_.id)
      // group readings in 1 second windows
      .timeWindow(Time.seconds(1))
      // compute average temperature using a user-defined function
      .apply(new TemperatureAverager)
    // print result stream to standard out
    avgTemp.print()
    // execute application
    env.execute("Compute average sensor temperature")
  }
}
/** User-defined WindowFunction to compute the average temperature of SensorReadings */
class TemperatureAverager extends WindowFunction[SensorReading, SensorReading, String, TimeWindow] {
  /** apply() is invoked once for each window */
  override def apply(
    sensorId: String,
    window: TimeWindow,
    vals: Iterable[SensorReading],
    out: Collector[SensorReading]): Unit = {
    // compute the average temperature
    val (cnt, sum) = vals.foldLeft((0, 0.0))((c, r) ⇒ (c._1 + 1, c._2 + r.temperature))
    val avgTemp = sum / cnt
    // emit a SensorReading with the average temperature
    out.collect(SensorReading(sensorId, window.getEnd, avgTemp))
  }
}
```

##### 程序结构

###### 创建程序执行环境

创建程序执行环境是编写Flink程序的首要步骤 程序执行环境是决定程序执行在本地还是在集群上

```scala
// set up the streaming execution environment
val env = StreamExecutionEnvironment.getExecutionEnvironment
// use event time for the application
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
// configure watermark interval
env.getConfig.setAutoWatermarkInterval(1000L) 
```

###### 读取数据流

StreamExecutionEnvironment类提供了从一个数据源中将数据读取到应用内的方法 数据流可以从消息队列 文件中生成

```scala
val sensorData: DataStream[SensorReading] = env.addSource(new SensorSource)
```

###### 转换数据流

```scala
val avgTemp: DataStream[SensorReading] = sensorData
     .map( r => {
           val celsius = (r.temperature - 32) * (5.0 / 9.0)
           SensorReading(r.id, r.timestamp, celsius)
       } )
     .keyBy(_.id)
     .timeWindow(Time.seconds(5))
     .apply(new TemperatureAverager)
```

###### 输出结果

```scala
avgTemp.print()
```

###### 执行

```scala
env.execute("Compute average sensor temperature")
```

#### 转换

##### 类型

| 转换类型 | 说明 |
| --- | ---- |
| Basic Transformations | 独立地对流中每个事件进行转换 |
| KeyedStream Transformations | 将流中的事件按键分组 对分组后的每组事件进行转换 |
| MultiStream transformations | 实现将多个流合并为一个流或切分一个流为多个流 |
|  Distribution Transformations| 实现重新组织流中的事件 |

##### Basic Transformations

| 转换算子 | 说明 | 接口 |
| ------- | ----- |  ----- |
| map  | 通过用户自定义映射函数对流中事件进行映射  |  // T: the type of input elements<br>// O: the type of output elements<br> MapFunction[T, O]<br>   &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;> map(T): O |
| filter | 通过用户自定义过滤函数对流中事件进行过滤 | // T: the type of input elements <br> // O: the type of output elementsFlatMapFunction[T, O] <br> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;> flatMap(T, Collector[O]): Unit |
| flatMap | 通过用户自定义扁平映射函数对流中事件进行扁平映射 | // T: the type of input elements <br>// O: the type of output elements FlatMapFunction[T, O] <br> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;> flatMap(T, Collector[O]): Unit|

##### KeyedStream Transformations

| 转换算子 | 说明 | 接口 |
| ------- | ----- | ------- |
| keyBy | 通过该转换通过在事件中指定一个字段为键可以将一个DataStream流转换为KeyedStream流 | val readings: DataStream[SensorReading] = ... <br> val keyed: KeyedStream[SensorReading, String] = readings.keyBy(r => r.id) |
| rolling aggregations | 该算子通过应用预定义聚集转换算子将KeyedStream流转换为DataStream流 如sum minimum和maximum | |
| reduce | 该算子通过应用自定义聚集算子将KeyedStream流转换为DataStream流 | // T: the element type <br> ReduceFunction[T] <br> &nbsp;&nbsp;&nbsp; > reduce(T, T): T|


##### MultiStream Transformations

| 转换算子 | 说明  | 代码 |
| ----- | ------- | -------|
| union  | 该算子实现合并多个独立具有相同类型的DataStream流为一个流 后续的转换算子将应用到所有被合并的流上 |```val parisStream: DataStream[SensorReading] = ... <br> val tokyoStream: DataStream[SensorReading] = ... <br>val allCities: DataStream[SensorReading] = parisStream.union(tokyoStream, rioStream)```|
| | | |
| split  | slit算子实现union算子相反的功能  其可以将一个输入流切分为多个相同类型的输出流 对每个输入事件其可以被发送给零个 一个或多个输出流 该算子接受OutputSelector实例其定义流事件如何被发送给特定的输出流 | val inputStream: DataStream[(Int, String)] = ... <br> val splitted: SplitStream[(Int, String)] = inputStream.split(t => if (t._1 > 1000) Seq("large") else Seq("small")) <br>val large: DataStream[(Int, String)] = splitted.select("large") <br> val small: DataStream[(Int, String)] = splitted.select("small") <br> val all: DataStream[(Int, String)] = splitted.select("small", "large")  |
##### Distribution Transformations

##### 并行度

##### 类型

#### 窗口算子

#### 状态算子

