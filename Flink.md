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

#### 窗口算子

#### 状态算子

