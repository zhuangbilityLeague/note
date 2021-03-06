# Spark Steaming DEMO

本例子使用 Kafka 作为数据采集来源，Spark Steaming 进行分析统计。

##环境

- 系统 ： centos 7
- Spark 包版本： spark-1.6.1-hadoop-2.6.0
- Scala 版本 scala 2.10.6
- IDE ： IDEA

## 步骤

1. 创建 SBT 工程

2.  引入依赖
	
	编辑 build.sbt 文件，在文件中追加如下代码。
	```sbt
	libraryDependencies ++= Seq(
		"org.apache.spark" % "spark-streaming_2.10" % "1.6.1",
		"org.apache.spark" % "spark-streaming-kafka_2.10" % "1.6.1"
	)
	```

3. 代码编写
	
	- 在 src/main/scala 目录下创建包 Stream
	- 在 Stream 包下创建 Scala 类文件 Kafka **并选择创建类型为 Object**

       示例代码如下
	```scala
	package Stream

	import kafka.serializer.StringDecoder
	import org.apache.spark.SparkConf
	import org.apache.spark.streaming.{Seconds, StreamingContext}
	import org.apache.spark.streaming.kafka._
	
	object Kafka {
	  def main(args: Array[String]) {
	
	    if(args.length < 2){
	      System.err.println("error")
	      System.exit(1)
	    }
	
	    val sparkConf = new SparkConf().setAppName("Kafka")
	    val ssc = new StreamingContext(sparkConf, Seconds(10))
	
	    ssc.checkpoint("checkPoint")
	
	    val Array(brokers,topics) = args
	
	    val topicsSet = topics.split(" ").toSet
	    val kafkaParams = Map[String,String]("metadata.broker.list" -> brokers)
	    val messages = KafkaUtils.createDirectStream[String,String,StringDecoder,StringDecoder](ssc,kafkaParams,topicsSet)
	    val lines = messages.map(_._2)
	
	    val words = lines.flatMap(_.split(" "))
	    val wordCounts = words.map(x => (x, 1L)).reduceByKey(_ + _)
	
	    wordCounts.print()
	
	    ssc.start()
	    ssc.awaitTermination()
	  }
	}	
	```

4. 编辑代码

	- 菜单栏进入 File > Project Structure > Artifacts
	- 选择 「+」 > JAR > From modules with dependencies
	-  选择 Main Class 为 Stream.Redis 后一路 OK 退回代码界面
	- 菜单栏进入 Build > Build Artifacts
	- 在弹出窗口中选刚才创建的 Artifact 选择 Build 操作并等待

5. 运行代码

	编译打包好的 Jar 位于项目根目录下 out/artifacts/「your-artifact-name」下
	运行如下代码即可提交任务至 spark 中执行
	```bash
	# 运行 zookeeper 服务
	/PATH/TO/KAFKA/bin/zookeeper-server-start.sh /PATH/TO/KAFKA/config/zookeeper.properties
	# 运行 kafka 服务
	/PATH/TO/KAFKA/bin/kafka-server-start.sh /PATH/TO/KAFKA/config/server.properties
	# 运行 spark 服务
	/PATH/TO/SPARK/sbin/start-all.sh
	# 提交任务
	/PATH/TO/SPARK/bin/spark-submit  --executor-memory 1g --master spark://127.0.0.1:7077 --class Stream.Kafka  [path/to/artifacts.jar] [kafka-address] [topic-name]	
	```
	
6. 向 Kafka 发送数据，查看结果