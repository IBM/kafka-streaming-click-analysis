## Clickstream Analysis using Apache Spark and Kafka

Clickstream analysis offers useful information about the usage characteristics of a website.

Popular use cases include:

1. <b>A/B Testing.</b> Statistically study how users of a web site are affected by changes from version A to B. [Read more](https://en.wikipedia.org/wiki/A/B_testing)

2. <b>Recommendation generation on shopping portals.</b> By determining the order in which a user clicks on a web site, what influenced him/her to a buying decision can be ascertained. This information can be used as a recommendation generation for future such patterns of clicks. 

3. <b>Targetted advertisement.</b> Simillar to recommendation generation, by tracking user clicks across websites, users of websites are served more precisely targetted advertisement.

4. <b>Trending topics.</b> Clickstream can be used to study or report trending topics in real time. For a particular time quantum, display top items that gets highest number of user clicks. 

In this journey, we choose to demonstrate, how to detect trending topics on 
wikipedia in real time. Kafka is used as a message queue and spark structured 
streaming is used for performing the analytics. This comnbination of kafka and
spark structured streaming is well known, for its usability and high throghput 
with low latency characterstics.

When you complete this journey, you will understand how to:

1. Perform clickstream analysis using, structured streaming.

2. Build a low latency stream processing, reading from kafka.

## Featured technologies

* [Apache Spark](http://spark.apache.org/): An open-source distributed computing framework that allows you to perform large-scale data processing.
* [Apache Kafka](http://kafka.apache.org) Kafka™ is used for building real-time data pipelines and streaming apps. It is designed to be horizontally scalable, fault-tolerant and fast.

![](doc/source/images/architecture.png)


  
## Set up instructions.

To set it up, please install kafka and Apache spark 2.2.0 on your system. 

In case an existing clickstream is not available for processing. A simulating clickstream can be used. 
An external publisher (simulating a real click stream) publishing to a topic `clicks`, on kafka running on <ip:port>, can be setup by 

1. first downloading the data from: [Wikipedia Clickstream data](https://meta.wikimedia.org/wiki/Research:Wikipedia_clickstream#Where_to_get_the_Data "Wikipedia clickstream data").

2. Kafka distribution comes with a handy command line utility for this purpose, once the data is downloaded and extracted.

Run,
```
$> tail -200 data/2017_01_en_clickstream.tsv | KAFKA_OPTS="-Djava.security.auth.login.config=config/jaas.conf" bin/kafka-console-producer.sh --broker-list ip:port  --topic clicks --producer.config=config/producer.properties
```

*Note: One can use unix head or tail utilities for conveniently specifying the range of rows to be sent for simulating clickstream.*

## Running locally using Spark shell.

### Bootstrap Spark.
Go to the spark install directory and bootstrap the spark shell specifying 
the correct version of spark and kafka.
```
$> cd $SPARK_DIR
$> bin/spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.11:2.2.0
```
### Setup Schema.

On the spark shell prompt, specify schema of the incoming wikipedia clickstream 
and parse method.

```scala
scala> import scala.util.Try
import scala.util.Try

scala> case class Click(prev: String, curr: String, link: String, n: Long)
defined class Click

scala> def parseVal(x: Array[Byte]): Option[Click] = {
    val split: Array[String] = new Predef.String(x).split("\\t")
    if (split.length == 4) {
      Try(Click(split(0), split(1), split(2), split(3).toLong)).toOption
    } else
      None
  }
       |      |      |      |      |      | parseVal: (x: Array[Byte])Option[Click]
```
### Setup structured streaming to read from Kafka.
```scala
scala> val records = spark.readStream.format("kafka")
                      .option("subscribe", "clicks")
                      .option("failOnDataLoss", "false")
                      .option("kafka.bootstrap.servers", "<ip:port>").load()
```

### Process records.
```scala
scala> 
    val messages = records.select("value").as[Array[Byte]]
                     .flatMap(x => parseVal(x))
                     .groupBy("curr")
                     .agg(Map("n" -> "sum"))
                     .sort($"sum(n)".desc)
```

### Output on console and start streaming.
```scala
val query = messages.writeStream
              .outputMode("complete")
              .option("truncate", "false")
              .format("console")
              .start()
```

```
scala> -------------------------------------------
Batch: 0

+---------------------------------------------+-------+
|curr                                         |sum(n) |
+---------------------------------------------+-------+
|Gavin_Rossdale                               |1205584|
|Unbreakable_(film)                           |1100870|
|Ben_Affleck                                  |939473 |
|Jacqueline_Kennedy_Onassis                   |926204 |
|Tom_Cruise                                   |743553 |
|Jackie_Chan                                  |625123 |
|George_Washington                            |622800 |
|Bill_Belichick                               |557286 |
|Mary,_Queen_of_Scots                         |547621 |
|The_Man_in_the_High_Castle                   |529446 |
|Clint_Eastwood                               |526275 |
|Beyoncé                                      |513177 |
|United_States_presidential_line_of_succession|490999 |
|Sherlock_Holmes                              |477874 |
|Winona_Ryder                                 |449984 |
|Titanic_(1997_film)                          |400197 |
|Watergate_scandal                            |381000 |
|Jessica_Biel                                 |379224 |
|Patrick_Swayze                               |373626 |
+---------------------------------------------+-------+
only showing top 20 rows

```

--------------------------------------------------------------

Resultant table shows the wikipedia pages with maximum hits. This table updates
automatically as soon as more data arrives from kafka. Unless specified otherwise,
structured streaming performs processing as soon as it sees some data.

Here we assume, the higher number of clicks indicate a "Hot topic" or "Trending topic".
Please feel free, to contribute more ideas on how to improve and even more type of
clickstream analytics that can be done.


## Running on IBM datascience experience.

Sign up for IBM datascience experience at [IBM DSX](https://datascience.ibm.com/). Once signed up, go ahead and create a project and add a notebook by loading from URL and provide [Notebook](notebooks/Clickstream_Analytics_using_Apache_Spark_and_Message_Hub.ipynb).

A message hub service can also be used incase, an existing kafka service is not available for testing. For this, one can add a data asset and create a message hub service. Later in the notebook credentials can be inserted using instructions given within [Notebook](notebooks/Clickstream_Analytics_using_Apache_Spark_and_Message_Hub.ipynb).


# License

[Apache 2.0](LICENSE)
