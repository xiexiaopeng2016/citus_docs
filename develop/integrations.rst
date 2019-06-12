外部集成
#####################

从Kafka摄入数据
=========================

例如，我们可以使用名为 `kafka-sink-pg-json <https://github.com/justonedb/kafka-sink-pg-json>`_ 的工具将JSON消息从Kafka主题复制到数据库表中。作为演示，我们将创建一个``kafka_test``表并从``test``主题中提取数据，并将JSON键自定义映射到表列。

使用Kafka进行实验的最简单方法是使用 `Confluent平台 <https://www.confluent.io/product/confluent-platform/>`_，其中包括Kafka，Zookeeper和相关工具，这些工具的版本经过验证可以协同工作。

.. code-block:: bash

  # we're using Confluent 2.0 for kafka-sink-pg-json support
  curl -L http://packages.confluent.io/archive/2.0/confluent-2.0.0-2.11.7.tar.gz \
    | tar zx

  # Now get the jar and conf files for kafka-sink-pg-json
  mkdir sink
  curl -L https://github.com/justonedb/kafka-sink-pg-json/releases/download/v1.0.2/justone-jafka-sink-pg-json-1.0.zip -o sink.zip
  unzip -d sink $_ && rm $_

下载的kafka-sink-pg-json包含一些配置文件。
我们想要连接到协调者Citus节点，因此我们必须编辑配置文件 ``sink/justone-kafka-sink-pg-json-connector.properties``：

.. code-block:: sh

  # add to sink/justone-kafka-sink-pg-json-connector.properties

  # the kafka topic we will use
  topics=test

  # db connection info
  # use your own settings here
  db.host=localhost:5432
  db.database=postgres
  db.username=postgres
  db.password=bar

  # the schema and table we will use
  db.schema=public
  db.table=kafka_test

  # the JSON keys, and columns to store them
  db.json.parse=/@a,/@b
  db.columns=a,b

通知 ``db.columns`` 和 ``db.json.parse``。这些列表的元素匹配，``db.json.parse`` 指定在传入的JSON对象中查找值的位置。

.. note::

  ``db.json.parse`` 路径以一种允许从JSON中灵活地获取值的语言编写。鉴于以下JSON，

  .. code-block:: json

    {
      "identity":71293145,
      "location": {
        "latitude":51.5009449,
        "longitude":-2.4773414
      },
      "acceleration":[0.01,0.0,0.0]
    }

  这里有一些示例路径以及它们匹配的内容：

  * ``/@identity`` - 元素71293145的路径。
  * ``/@location/@longitude`` - 元素-2.4773414的路径。
  * ``/@acceleration/#0`` - 元素0.01的路径
  * ``/@location`` - 元素``{"latitude":51.5009449, "longitude":-2.4773414}``的路径

我们自己的情况很简单。我们的事件将是像 ``{"a":1, "b":2}``。解析器将这些值拉入同名列。

设置好配置文件之后，就可以准备数据库了。使用psql连接到协调器节点并运行：

.. code-block:: psql

  -- create metadata tables for kafka-sink-pg-json
  \i sink/install-justone-kafka-sink-pg-1.0.sql

  -- create and distribute target ingestion table
  create table kafka_test ( a int, b int );
  select create_distributed_table('kafka_test', 'a');

启动Kafka机器：

.. code-block:: bash

  # save some typing
  export C=confluent-2.0.0

  # start zookeeper
  $C/bin/zookeeper-server-start \
    $C/etc/kafka/zookeeper.properties

  # start kafka server
  $C/bin/kafka-server-start \
    $C/etc/kafka/server.properties

  # create the topic we'll be reading/writing
  $C/bin/kafka-topics --create --zookeeper localhost:2181   \
                      --replication-factor 1 --partitions 1 \
                      --topic test

运行摄取程序：

.. code-block:: bash

  # the jar files for this are in "sink"
  export CLASSPATH=$PWD/sink/*

  # Watch for new events in topic and insert them
  $C/bin/connect-standalone \
    sink/justone-kafka-sink-pg-json-standalone.properties \
    sink/justone-kafka-sink-pg-json-connector.properties

此时Kafka-Connect正在监视test主题，并将在那里解析事件并将其插入 ``kafka_test``。让我们从命令行发送一个事件。

.. code-block:: bash

  echo '{"a":42,"b":12}' | \
    $C/bin/kafka-console-producer --broker-list localhost:9092 --topic test

在一小段延迟之后，新行将显示在数据库中。

::

  select * from kafka_test;

  ┌────┬────┐
  │ a  │ b  │
  ├────┼────┤
  │ 42 │ 12 │
  └────┴────┘

警告
-------

* 在撰写本文时，kafka-sink-pg-json需要Kafka 0.9或更早版本。
* kafka-sink-pg-json连接器配置文件不提供连接SSL支持的方法，因此该工具不适用于需要安全连接的Citus Cloud。
* Kafka主题中格式不正确的JSON字符串将导致工具卡住。需要手动干预主题以处理更多事件。

从Spark中提取数据
=========================

人们有时使用Spark来转换Kafka数据，比如通过添加计算值。在本节中，我们将了解如何将Spark数据帧摄取到分布式Citus表中。

首先让我们启动一个本地Spark集群。它有几个移动部件，所以最简单的方法是使用docker-compose运行这些部件。

.. code-block:: bash

  wget https://raw.githubusercontent.com/gettyimages/docker-spark/master/docker-compose.yml

  # this may require "sudo" depending on the docker daemon configuration
  docker-compose up

要摄取到PostgreSQL，我们将编写自定义Scala代码。我们将使用Scala构建工具（SBT）加载依赖项并运行我们的代码，因此请下载SBT并将其安装在您的计算机上。

接下来为我们的项目创建一个新目录。

.. code-block:: bash

  mkdir sparkcitus

创建一个名为 ``sparkcitus/build.sbt`` 告诉SBT我们的项目配置的文件，并添加：

.. code-block:: scala

  // add this to build.sbt

  name := "sparkcitus"
  version := "1.0"

  scalaVersion := "2.10.4"

  resolvers ++= Seq(
    "Maven Central" at "http://central.maven.org/maven2/"
  )

  libraryDependencies ++= Seq(
    "org.apache.spark" %% "spark-core" % "2.2.1",
    "org.apache.spark" %% "spark-sql"  % "2.2.1",
    "org.postgresql"   %  "postgresql" % "42.2.2"
  )

接下来创建一个帮助器Scala类，用于通过JDBC进行提取。将以下内容添加到 ``sparkcitus/copy.scala``：

.. code-block:: scala

  import java.io.InputStream
  import java.sql.DriverManager
  import java.util.Properties

  import org.apache.spark.sql.{DataFrame, Row}
  import org.postgresql.copy.CopyManager
  import org.postgresql.core.BaseConnection

  object CopyHelper {

    def rowsToInputStream(rows: Iterator[Row]): InputStream = {
      val bytes: Iterator[Byte] = rows.map { row =>
        (row.toSeq
          .map { v =>
            if (v == null) {
              """\N"""
            } else {
              "\"" + v.toString.replaceAll("\"", "\"\"") + "\""
            }
          }
          .mkString("\t") + "\n").getBytes
      }.flatten

      new InputStream {
        override def read(): Int =
          if (bytes.hasNext) {
            bytes.next & 0xff // make the signed byte an unsigned int
          } else {
            -1
          }
      }
    }

    def copyIn(url: String, df: DataFrame, table: String):Unit = {
      var cols = df.columns.mkString(",")

      df.foreachPartition { rows =>
        val conn = DriverManager.getConnection(url)
        try {
          val cm = new CopyManager(conn.asInstanceOf[BaseConnection])
          cm.copyIn(
            s"COPY $table ($cols) " + """FROM STDIN WITH (NULL '\N', FORMAT CSV, DELIMITER E'\t')""",
            rowsToInputStream(rows))
          ()
        } finally {
          conn.close()
        }
      }
    }
  }

继续设置，将一些样本数据保存到 ``people.json``。注意故意缺少周围的方括号。稍后我们将从数据中创建Spark数据帧。

.. code-block:: js

  {"name":"Tanya Rosenau"   , "age": 24},
  {"name":"Rocky Slay"      , "age": 85},
  {"name":"Tama Erdmann"    , "age": 48},
  {"name":"Jared Olivero"   , "age": 42},
  {"name":"Gudrun Shannon"  , "age": 53},
  {"name":"Quentin Yoon"    , "age": 32},
  {"name":"Yanira Huckstep" , "age": 53},
  {"name":"Brendon Wesley"  , "age": 19},
  {"name":"Minda Nordeen"   , "age": 79},
  {"name":"Katina Woodell"  , "age": 83},
  {"name":"Nevada Mckinnon" , "age": 65},
  {"name":"Georgine Mcbee"  , "age": 56},
  {"name":"Mittie Vanetten" , "age": 17},
  {"name":"Lecia Boyett"    , "age": 37},
  {"name":"Tobias Mickel"   , "age": 69},
  {"name":"Jina Mccook"     , "age": 82},
  {"name":"Cassidy Turrell" , "age": 37},
  {"name":"Cherly Skalski"  , "age": 29},
  {"name":"Reita Bey"       , "age": 69},
  {"name":"Keely Symes"     , "age": 34}

最后，在Citus中创建和分布一个表

.. code-block:: sql

  create table spark_test ( name text, age integer );
  select create_distributed_table('spark_test', 'name');

现在我们已经准备好将所有东西挂钩了。启动sbt：

.. code-block:: bash

  # run this in the sparkcitus directory

  sbt

进入sbt后，编译项目，然后进入“控制台”，这是一个加载我们的代码和依赖项的Scala repl：

.. code-block:: text

  sbt:sparkcitus> compile
  [success] Total time: 3 s

  sbt:sparkcitus> console
  [info] Starting scala interpreter...

  scala> 

在控制台中键入以下Scala命令：

.. code-block:: scala

  // inside the sbt scala interpreter

  import org.apache.spark.sql.SparkSession

  // open a session to the Spark cluster
  val spark = SparkSession.builder().appName("sparkcitus").config("spark.master", "local").getOrCreate()

  // load our sample data into Spark
  val df = spark.read.json("people.json")

  // this is a simple connection url (it assumes Citus
  // is running on localhost:5432), but more complicated
  // JDBC urls differ subtly from Postgres urls, see:
  // https://jdbc.postgresql.org/documentation/head/connect.html
  val url = "jdbc:postgresql://localhost/postgres"

  // ingest the data frame using our CopyHelper class
  CopyHelper.copyIn(url, df, "spark_test")

这使用CopyHelper来摄取信息。此时，数据将出现在分布式表中。

.. note::

  Our method of ingesting the dataframe is straightforward but doesn't protect against Spark errors. Spark guarantees "at least once" semantics, i.e. a read error can cause a subsequent read to encounter previously seen data.

  A more complicated, but robust, approach is to use the custom Spark partitioner `spark-citus <https://github.com/koeninger/spark-citus>`_ so that partitions match up exactly with Citus shards. This allows running transactions directly on worker nodes which can rollback on read failure. See the presentation linked in that repository for more information.

Business Intelligence with Tableau
==================================

`Tableau <https://www.tableau.com/>`_ is a popular business intelligence and analytics tool for databases. Citus and Tableau provide a seamless experience for performing ad-hoc reporting or analysis.

You can now interact with Tableau using the following steps.

* Choose PostgreSQL from the "Add a Connection" menu.

  .. image:: ../images/tableau-add-connection.png
* Enter the connection details for the coordinator node of your Citus cluster. (Note if you're connecting to Citus Cloud you must select "Require SSL.")

  .. image:: ../images/tableau-connection-details.png
* Once you connect to Tableau, you will see the tables in your database. You can define your data source by dragging and dropping tables from the “Table” pane. Or, you can run a custom query through “New Custom SQL”.
* You can create your own sheets by dragging and dropping dimensions, measures, and filters. You can also create an interactive user interface with Tableau. To do this, Tableau automatically chooses a date range over the data. Citus can compute aggregations over this range in human real-time.

.. image:: ../images/tableau-visualization.jpg
