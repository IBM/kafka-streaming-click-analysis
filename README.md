# Clickstream Analysis using Apache Spark and Apache Kafka

Clickstream analysis is the process of collecting, analyzing, and reporting about which web pages a user visits, and can offer useful information about the usage characteristics of a website.

Some popular use cases for clickstream analysis include:

* **A/B Testing:** Statistically study how users of a web site are affected by changes from version A to B. [Read more](https://en.wikipedia.org/wiki/A/B_testing)

* **Recommendation generation on shopping portals:** Click patterns of users of a shopping portal website, indicate how a user was influenced into buying something. This information can be used as a recommendation generation for future such patterns of clicks.

* **Targeted advertisement:** Similar to *recommendation generation*, but tracking user clicks "across websites" and using that information to target advertisement in real-time.

* **Trending topics:** Clickstream can be used to study or report trending topics in real time. For a particular time quantum, display top items that gets the highest number of user clicks.

In this Code Pattern, we will demonstrate how to detect real-time trending topics on the [Wikipedia](https://www.wikipedia.org/) web site. To perform this task, Apache Kafka will be used as a message queue, and the Apache Spark structured streaming engine will be used to perform the analytics. This combination is well known for its usability, high throughput and low-latency characteristics.

When you complete this Code Pattern, you will understand how to:

* Use [Jupyter Notebooks](https://jupyter.org/) to load, visualize, and analyze data
* Run streaming analytics interactively using Notebooks in [IBM Watson Studio](https://dataplatform.cloud.ibm.com/)
* Interactively develop clickstream analysis using Apache Spark Structured Streaming on a Spark Shell
* Build a low-latency processing stream utilizing Apache Kafka.

## Flow

![architecture](doc/source/images/architecture.png)

1. User connects with Apache Kafka service and sets up a running instance of a clickstream.
2. Run a Jupyter Notebook in IBM's Watson Studio that interacts with the underlying Apache Spark service. Alternatively, this can be done locally by running the Spark Shell.
3. The Spark service reads and processes data from the Kafka service.
4. Processed Kafka data is relayed back to the user via the Jupyter Notebook (or console sink if running locally).

## Included components

* [IBM Watson Studio](https://www.ibm.com/cloud/watson-studio): Analyze data using RStudio, Jupyter, and Python in a configured, collaborative environment that includes IBM value-adds, such as managed Spark.
* [Apache Spark](https://spark.apache.org/): An open-source distributed computing framework that allows you to perform large-scale data processing.
* [Apache Kafka](https://kafka.apache.org): Kafka is used for building real-time data pipelines and streaming apps. It is designed to be horizontally scalable, fault-tolerant and fast.
* [Jupyter Notebook](https://jupyter.org/): An open source web application that allows you to create and share documents that contain live code, equations, visualizations, and explanatory text.
* [Event Streams](https://cloud.ibm.com/catalog/services/event-streams): A scalable, high-throughput message bus. Wire micro-services together using open protocols.

## Watch the Video

[![video](https://img.youtube.com/vi/-3QY1gT5oao/0.jpg)](https://www.youtube.com/watch?v=-3QY1gT5oao)

## Steps

There are two modes of exercising this Code Pattern:

* [Run locally using the Spark shell](#run-locally) OR
* [Run with IBM Watson Studio](#run-with-ibm-watson-studio)

> **NOTE**: Running with Watson Studio will require an [Event Streams](https://cloud.ibm.com/catalog/services/event-streams) service, which is not free.

## Run locally

1. [Install Spark and Kafka](#1-install-spark-and-kafka)
2. [Setup and run a simulated clickstream](#2-setup-and-run-a-simulated-clickstream)
3. [Run the script](#3-run-the-script)

### 1. Install Spark and Kafka

Install by downloading and extracting a binary distribution from [Apache Kafka](https://kafka.apache.org/downloads) (0.10.2.1 is the recommended version) and [Apache Spark 2.2.0](https://spark.apache.org/releases/spark-release-2-2-0.html) on your system.

### 2. Setup and run a simulated clickstream

> **NOTE**: These steps can be skipped if you already have a clickstream available for processing. If so, create and stream data to the topic named 'clicks' before proceeding to the next step.

Use the following steps to setup a simulation clickstream that uses data from an external publisher:

1. Download and extract the [Wikipedia Clickstream](https://figshare.com/articles/Wikipedia_Clickstream/1305770). Select any data set, the set [`2017_01_en_clickstream.tsv.gz`](https://ndownloader.figshare.com/files/7563832) was used for this Code Pattern.

2. Create and run a local Kafka service instance by following the instructions listed in the [Kafka Quickstart Documentation](https://kafka.apache.org/quickstart). Be sure to create a topic named `clicks`.

3. The Kafka distribution comes with a handy command line utility for uploading data to the Kafka service. To process the simulated Wikipedia data, run the following commands:

> **NOTE**: Replace `ip:port` with the correct values of the running Kafka service, which is defaulted to `localhost:9092` when running locally.

```bash
cd kafka_2.10-0.10.2.1
tail -200 data/2017_01_en_clickstream.tsv | bin/kafka-console-producer.sh --broker-list ip:port --topic clicks --producer.config=config/producer.properties
```

> **TIP**: Unix head or tail utilities can be used for conveniently specifying the range of rows to be sent for simulating the clickstream.

### 3. Run the script

Go to the Spark install directory and bootstrap the Spark shell specifying the correct versions of Spark and Kafka:

```bashh
cd $SPARK_DIR
bin/spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.11:2.2.0
```

In the spark shell prompt, specify the schema of the incoming wikipedia clickstream and parse method:

> **TIP**: For conveniently copying and pasting commands into the spark shell, spark-shell supports a `:paste` mode*

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

> **NOTE**: Replace `ip:port` with the correct values of ip and port of the running Kafka service, which is defaulted to `localhost:9092` when running locally.

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

Output to the console and start streaming data (using the `tail` clickstream command descibed above):

```scala
val query = messages.writeStream
              .outputMode("complete")
              .option("truncate", "false")
              .format("console")
              .start()
```

```scala
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

The resultant table shows the Wikipedia pages that had the most hits. This table updates automatically whenever more data arrives from Kafka. Unless specified otherwise, structured streaming performs processing as soon as it sees any data.

Here we assume the higher number of clicks indicates a "Hot topic" or "Trending topic". Please feel free to contribute any ideas on how to improve this, or thoughts on any other types of clickstream analytics that can be done.

## Run with IBM Watson Studio

1. [Create a new Watson Studio project](#1-create-a-new-watson-studio-project)
2. [Create the notebook](#2-create-the-notebook)
3. [Set up Even Streams](#3-set-up-event-streams)
4. [Run the notebook](#4-run-the-notebook)
5. [Upload data](#5-upload-data)

> **NOTE**: Running with Watson Studio will require an [Event Streams](https://cloud.ibm.com/catalog/services/event-streams) service, which is not free.

### 1. Create a new Watson Studio project

* Log into IBM's [Watson Studio](https://dataplatform.cloud.ibm.com). Once in, you'll land on the dashboard.

* Create a new project by clicking `+ New project` and choosing `Data Science`:

  ![studio project](https://raw.githubusercontent.com/IBM/pattern-utils/master/watson-studio/new-project-data-science.png)

* Enter a name for the project name and click `Create`.

* **NOTE**: By creating a project in Watson Studio a free tier `Object Storage` service and `Watson Machine Learning` service will be created in your IBM Cloud account. Select the `Free` storage type to avoid fees.

  ![studio-new-project](https://raw.githubusercontent.com/IBM/pattern-utils/master/watson-studio/new-project-data-science-name.png)

* Upon a successful project creation, you are taken to a dashboard view of your project. Take note of the `Assets` and `Settings` tabs, we'll be using them to associate our project with any external assets (datasets and notebooks) and any IBM cloud services.

  ![studio-project-dashboard](https://raw.githubusercontent.com/IBM/pattern-utils/master/watson-studio/overview-empty.png)

### 2. Create the notebook

* From the new project `Overview` panel, click `+ Add to project` on the top right and choose the `Notebook` asset type.

  ![studio-project-dashboard](https://raw.githubusercontent.com/IBM/pattern-utils/master/watson-studio/add-assets-notebook.png)

* Fill in the following information:

  * Select the `From URL` tab. [1]
  * Enter a `Name` for the notebook and optionally a description. [2]
  * Under `Notebook URL` provide the following url: [https://raw.githubusercontent.com/IBM/kafka-streaming-click-analysis/master/notebooks/Clickstream_Analytics_using_Apache_Spark_and_Message_Hub.ipynb](https://raw.githubusercontent.com/IBM/kafka-streaming-click-analysis/master/notebooks/Clickstream_Analytics_using_Apache_Spark_and_Message_Hub.ipynb) [3]
  * For `Runtime` select the `Spark Python 3.6` option. [4]

  ![add notebook](https://github.com/IBM/pattern-utils/raw/master/watson-studio/notebook-create-url-spark-py36.png)

* Click the `Create` button.

* **TIP:** Once successfully imported, the notebook should appear in the `Notebooks` section of the `Assets` tab.

### 3. Set up Event Streams

Before running the notebook, you will need to setup a [Event Streams](https://cloud.ibm.com/catalog/services/event-streams) service.

* To create a Event Streams service, go to the `Data Services` -> `Services` tab on the IBM Watson Studio dashboard. Click `Create`, then select the Event Streams service. Select the `Standard` plan then follow the on-screen instructions to create the service. Once created, select the Event Streams service instance to bring up the details panel where you can create a topic. In the create form, name the topic `clicks` and leave the other fields with their default values.

* Next create a connection to this service so that it can be added as an asset to the project. Go to the `Data Services` -> `Connections` tab on the Watson Studio dashboard. Click `Create New` to create a connection. Provide a unique name and then select the just created Event Streams instance as the `Service Instance` connection.

* Next attach the connection as an asset to the project. Go to the `Assets` tab on your project dashboard. Click on `Add to project` and select the `Data Asset` option. Then click on the `Connections` tab and select your just created connection. Click 'Apply' to add the connection.

### 4. Run the notebook

* Click the `(►) Run` button to start stepping through the notebook.

* When you approach the section entitled `Credentials Section` stop executing the cells and update the credentials. We do this by inserting credentials for the Event Streams connection you just created.

* Then click on the `1001` button located in the top right corner of the notebook. Select the `Connections` tab to see your Event Streams connector. Click the `Insert to code` button to download the Event Streams credentials data into code cell `[1]`.

> **NOTE**: Make sure you rename the credentials object to `credentials_1`.

* Now finish the notebook by running each cell one at a time.

#### Tips on running a notebook

When a notebook is executed, what is actually happening is that each code cell in the notebook is executed, in order, from top to bottom.

Each code cell is selectable and is preceded by a tag in the left margin. The tag format is `In [x]:`. Depending on the state of the notebook, the `x` can be:

* A blank, this indicates that the cell has never been executed.
* A number, this number represents the relative order this code step was executed.
* A `*`, this indicates that the cell is currently executing.

### 5. Upload data

For uploading data to the Event Streams or Apache Kafka as a service, use the kafka command line utility. Using the detailed instructions found in the [Setup and run a simulated clickstream](#2-setup-and-run-a-simulated-clickstream) section above, you need to:

1. Download the Wikipedia data.
2. Download the Kafka distribution binary.
3. Download [config/messagehub.properties](config/messagehub.properties) config file and update message hub credentials, found in the credentials section of the notebook.

> **NOTE**: Ignore extra set of double quotes in the password (if any), while copying it.

After downloading and extracting the Kafka distribution binary and the data, run the command as follows:

> **NOTE**: Replace `ip:port` with the `kafka_brokers_sasl` value found in the credentials section of the notebook, described in previous step.

```bash
cd kafka_2.10-0.10.2.1
tail -200 data/2017_01_en_clickstream.tsv | bin/kafka-console-producer.sh --broker-list ip:port --request-timeout-ms 30000 --topic clicks --producer.config=config/messagehub.properties
```

## Learn more

* **Data Analytics Code Patterns**: Enjoyed this Code Pattern? Check out our other [Data Analytics Code Patterns](https://developer.ibm.com/technologies/data-science/)
* **AI and Data Code Pattern Playlist**: Bookmark our [playlist](https://www.youtube.com/playlist?list=PLzUbsvIyrNfknNewObx5N7uGZ5FKH0Fde) with all of our Code Pattern videos
* **Watson Studio**: Master the art of data science with IBM's [Watson Studio](https://dataplatform.cloud.ibm.com/)

## License

This code pattern is licensed under the Apache Software License, Version 2.  Separate third party code objects invoked within this code pattern are licensed by their respective providers pursuant to their own separate licenses. Contributions are subject to the [Developer Certificate of Origin, Version 1.1 (DCO)](https://developercertificate.org/) and the [Apache Software License, Version 2](https://www.apache.org/licenses/LICENSE-2.0.txt).

[Apache Software License (ASL) FAQ](https://www.apache.org/foundation/license-faq.html#WhatDoesItMEAN)
