# Clickstream Analysis using Apache Spark and Apache Kafka

Clickstream analysis is the process of collecting, analzing, and reporting about which web pages a user visits, and can offer useful information about the usage characteristics of a website.

Some popular use cases for clickstream analysis include:

* <b>A/B Testing.</b> Statistically study how users of a web site are affected by changes from version A to B. [Read more](https://en.wikipedia.org/wiki/A/B_testing)

* <b>Recommendation generation on shopping portals.</b> Click patterns of users of a shopping portal website, indicate how a user was influenced into buying something. This information can be used as a recommendation generation for future such patterns of clicks.

* <b>Targeted advertisement.</b> Similar to <i>recommendation generation</i>, but tracking user clicks across websites and using that information to target advertisement in real-time and more accurately.

* <b>Trending topics.</b> Clickstream can be used to study or report trending topics in real time. For a particular time quantum, display top items that gets the highest number of user clicks.

In this Code Pattern, we will demonstrate how to detect real-time trending topics on the [Wikipedia](https://www.wikipedia.org/) web site. To perform this task, Apache Kafka will be used as a message queue, and the Apache Spark structured streaming engine will be used to perform the analytics. This combination is well known for its usability, high throughput and low-latency characteristics.

When you complete this Code Pattern, you will understand how to:

* Perform clickstream analysis using Apache Spark Structured Streaming.
* Build a low-latency processing stream utilizing Apache Kafka.

![](doc/source/images/architecture.png)

## Flow

1. A running instance of a clickstream, over an Apache Kafka service.
2. Apache Spark reads the clickstream from a specified topic of an Apache Kafka service.
3. The notebook is run using IBM's Data Science Experience (or run locally).
4. The processed stream is printed to console.

## Included components

* [Apache Spark](http://spark.apache.org/): An open-source distributed computing framework that allows you to perform large-scale data processing.
* [Apache Kafka](http://kafka.apache.org): Kafka is used for building real-time data pipelines and streaming apps. It is designed to be horizontally scalable, fault-tolerant and fast.

# Steps

There are two modes of exercising this Code Pattern:
* [Run locally using the Spark shell](#run-locally).
* [Run using a Jupyter notebook in the IBM Data Science Experience](#run-using-a-jupyter-notebook-in-the-ibm-data-science-experience). *Note: Running in this mode  requires a [Message Hub](https://developer.ibm.com/messaging/message-hub/) service, which charges a nominal fee.*

## Run locally
1. [Install Spark and Kafka](#1-install-spark-and-kafka)
2. [Setup and run a simulated clickstream](#2-setup-and-run-a-simulated-clickstream)
3. [Run the script](#3-run-the-script)

### 1. Install Spark and Kafka

Install by downloading and extracting a binary distribution from [Apache Kafka](https://kafka.apache.org/downloads) (0.10.2.1 is the recommended version) and [Apache Spark 2.2.0](https://spark.apache.org/releases/spark-release-2-2-0.html) on your system.

### 2. Setup and run a simulated clickstream

*Note: These steps can be skipped if you already have a clickstream available for processing. If so, create and stream data to the topic named 'clicks' before proceeding to the next step.*

Use the following steps to setup a simulation clickstream that uses data from an external publisher:

1. Download and extract the `Wikipedia Clickstream` data from [here](https://meta.wikimedia.org/wiki/Research:Wikipedia_clickstream#Where_to_get_the_Data "Wikipedia clickstream data"). Select the data set that was used to test this Code Pattern -  `2017_01_en_clickstream.tsv.gz`.

2. Create and run a local Kafka service instance by following the instructions listed [here](http://kafka.apache.org/quickstart). Be sure to create a topic named `clicks`.

3. The Kafka distribution comes with a handy command line utility for uploading data to the Kafka service. To process the simulated Wikipedia data, run the following commands:

*Note: Replace `ip:port` with the correct values of the running Kafka service, which is defaulted to `localhost:2181` when running locally.*

```
$ cd kafka_2.10-0.10.2.1
$ tail -200 data/2017_01_en_clickstream.tsv | bin/kafka-console-producer.sh --broker-list ip:port --topic clicks --producer.config=config/producer.properties
```

*Tip: Unix head or tail utilities can be used for conveniently specifying the range of rows to be sent for simulating the clickstream.*

### 3. Run the script

Go to the Spark install directory and bootstrap the Spark shell specifying the correct versions of Spark and Kafka:

```
$ cd $SPARK_DIR
$ bin/spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.11:2.2.0
```

In the spark shell prompt, specify the schema of the incoming wikipedia clickstream and parse method:

*Tip: For conveniently copying and pasting commands into the spark shell, spark-shell supports a `:paste` mode*

```scala
scala> import scala.util.Try

scala> case class Click(prev: String, curr: String, link: String, n: Long)

scala> def parseVal(x: Array[Byte]): Option[Click] = {
    val split: Array[String] = new Predef.String(x).split("\\t")
    if (split.length == 4) {
      Try(Click(split(0), split(1), split(2), split(3).toLong)).toOption
    } else
      None
  }

```

Setup structured streaming to read from Kafka:

*Note: Replace `ip:port` with the correct values of ip and port of the running Kafka service, which is defaulted to `localhost:2181` when running locally.*

```scala
scala> val records = spark.readStream.format("kafka")
                      .option("subscribe", "clicks")
                      .option("failOnDataLoss", "false")
                      .option("kafka.bootstrap.servers", "ip:port").load()
```

Process the records:

```scala
scala>
    val messages = records.select("value").as[Array[Byte]]
                     .flatMap(x => parseVal(x))
                     .groupBy("curr")
                     .agg(Map("n" -> "sum"))
                     .sort($"sum(n)".desc)
```

Output to the console and start streaming:

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

The resultant table shows the Wikipedia pages that had the most hits. This table updates automatically whenever more data arrives from Kafka. Unless specified otherwise, structured streaming performs processing as soon as it sees any data.

Here we assume the higher number of clicks indicates a "Hot topic" or "Trending topic". Please feel free to contribute any ideas on how to improve this, or thoughts on any other types of clickstream analytics that can be done.

## Run using a Jupyter notebook in the IBM Data Science Experience

1. [Sign up for the Data Science Experience](#1-sign-up-for-the-data-science-experience)
2. [Create the notebook](#2-create-the-notebook)
3. [Run the notebook](#3-run-the-notebook)
4. [Upload data](#4-upload-data)
5. [Save and Share](#5-save-and-share)

### 1. Sign up for the Data Science Experience

Sign up for IBM's [Data Science Experience](http://datascience.ibm.com/). By signing up for the Data Science Experience, two services: ``DSX-Spark`` and ``DSX-ObjectStore`` will be created in your IBM Cloud account. If these services do not exist, or if you are already using them for some other application, you will need to create new instances.

To create these services:
* Login to your [IBM Cloud](http://bluemix.net) account.
* Create your Spark service by selecting the service type [Apache Spark](https://console.bluemix.net/catalog/services/apache-spark). If not already used, name your service ``DSX-Spark``.
* Create your Object Storage service by selecting the service type [Cloud Object Storage](https://console.bluemix.net/catalog/infrastructure/object-storage-group). If not already used, name your service ``DSX-ObjectStorage``.

> Note: When creating your Object Storage service, select the ``Swift`` storage type in order to avoid having to pay an upgrade fee.

Take note of your service names as you will need to select them in the following steps.

### 2. Create the notebook

First you must create a new Project:
* From the [IBM Data Science Experience page](https://apsportal.ibm.com/analytics) either click the ``Get Started`` tab at the top or scroll down to ``Recently updated projects``.
* Click on ``New project`` under ``Recently updated projects``.
* Enter a ``Name`` and optional ``Description``.
* For ``Spark Service``, select your Apache Spark service name.
* For ``Storage Type``, select the ``Object Storage (Swift API)`` option.
* For ``Target Object Storage Instance``, select your Object Storage service name.
* Click ``Create``.

![](doc/source/images/create-project.png)

Create the Notebook:
* Click on your project to open up the project details panel.
* Click ``add notebooks``.
* Click the tab for ``From URL`` and enter a ``Name`` and optional ``Description``.
* For ``Notebook URL`` enter: https://raw.githubusercontent.com/IBM/kafka-streaming/master/notebooks/Clickstream_Analytics_using_Apache_Spark_and_Message_Hub.ipynb
* For ``Spark Service``, select your Apache Spark service name.
* Click ``Create Notebook``.

![](doc/source/images/create-notebook.png)

### 3. Run the notebook

Before running the notebook, you will need to setup a [Message Hub](https://developer.ibm.com/messaging/message-hub/) service.

**Note:** Message Hub is a paid service.

* For creating a Message Hub service, go to `Data services-> Services` tab on the dashboard. Select the option to create a Message Hub service and follow the on-screen instructions. Post creating service instance, select it and create a topic `clicks` with defaults.

* Once the service is running, it has to be added to the current notebook. For this, first we need to create a connection for this Message Hub service instance. Go to `Data services-> connections` tab on dashboard and create new connection filling in the details and referring to the above created service instance in the Service instance section and then select topic `clicks`. Once done, go to `Assets` tab on the project dashboard and then click `+New data asset`. Then locate the created Message Hub service connection under the connections tab and click `Apply`.

* Once the service is added to the notebook, credentials to access it can be auto inserted. Please follow the comments found in the loaded notebook for instructions on how to insert the credentials. Once the credentials are inserted it is ready for execution.

When a notebook is executed, what is actually happening is that each code cell in
the notebook is executed, in order, from top to bottom.

Each code cell is selectable and is preceded by a tag in the left margin. The tag
format is `In [x]:`. Depending on the state of the notebook, the `x` can be:

* A blank, this indicates that the cell has never been executed.
* A number, this number represents the relative order this code step was executed.
* A `*`, this indicates that the cell is currently executing.

There are several ways to execute the code cells in your notebook:

* One cell at a time.
  * Select the cell, and then press the `Play` button in the toolbar.
* Batch mode, in sequential order.
  * From the `Cell` menu bar, there are several options available. For example, you
    can `Run All` cells in your notebook, or you can `Run All Below`, that will
    start executing from the first cell under the currently selected cell, and then
    continue executing all cells that follow.
* At a scheduled time.
  * Press the `Schedule` button located in the top right section of your notebook
    panel. Here you can schedule your notebook to be executed once at some future
    time, or repeatedly at your specified interval.


### 4. Upload data

For uploading data to the [Message Hub](https://developer.ibm.com/messaging/message-hub/) or Apache Kafka as a service, use the kafka command line utility. Using the detailed instructions found in the [Setup and run a simulated clickstream](#2-setup-and-run-a-simulated-clickstream) section above, you need to:

1) Download the Wikipedia data.
2) Download the Kafka distribution binary.

After downloading and extracting the Kafka distribution binary and the data, run the command as follows:

*Note: Replace `ip:port` with the broker urls found in the credentials section of the Message Hub service.*

```
$ cd kafka_2.10-0.10.2.1
$ tail -200 data/2017_01_en_clickstream.tsv | KAFKA_OPTS="-Djava.security.auth.login.config=config/jaas.conf" bin/kafka-console-producer.sh --broker-list ip:port --topic clicks --producer.config=config/producer.properties

```
**Note:** You might need to add credential information to the `jaas.conf` config file. It typical has the following format:

```
KafkaClient {
    org.apache.kafka.common.security.plain.PlainLoginModule required
	username="***"
	password="*****";
};

```
*Substitute the values of `username` and `password` from the inserted `credentials` section of the notebook, found in the previous step.*

### 5. Save and Share

#### How to save your work:

Under the `File` menu, there are several ways to save your notebook:

* `Save` will simply save the current state of your notebook, without any version
  information.
* `Save Version` will save your current state of your notebook with a version tag
  that contains a date and time stamp. Up to 10 versions of your notebook can be
  saved, each one retrievable by selecting the `Revert To Version` menu item.

#### How to share your work:

You can share your notebook by selecting the “Share” button located in the top
right section of your notebook panel. The end result of this action will be a URL
link that will display a “read-only” version of your notebook. You have several
options to specify exactly what you want shared from your notebook:

* `Only text and output`: will remove all code cells from the notebook view.
* `All content excluding sensitive code cells`:  will remove any code cells
  that contain a *sensitive* tag. For example, `# @hidden_cell` is used to protect
  your dashDB credentials from being shared.
* `All content, including code`: displays the notebook as is.
* A variety of `download as` options are also available in the menu.

# Learn more

* **Data Analytics Code Patterns**: Enjoyed this Code Pattern? Check out our other [Data Analytics Code Patterns](https://developer.ibm.com/code/technologies/data-science/)
* **AI and Data Code Pattern Playlist**: Bookmark our [playlist](https://www.youtube.com/playlist?list=PLzUbsvIyrNfknNewObx5N7uGZ5FKH0Fde) with all of our Code Pattern videos
* **Data Science Experience**: Master the art of data science with IBM's [Data Science Experience](https://datascience.ibm.com/)
* **Spark on IBM Cloud**: Need a Spark cluster? Create up to 30 Spark executors on IBM Cloud with our [Spark service](https://console.bluemix.net/catalog/services/apache-spark)

# License
[Apache 2.0](LICENSE)
