![OpenSearch logo](OpenSearch.svg)

# OpenSearch Hadoop
OpenSearch real-time search and analytics natively integrated with Hadoop.
Supports [Map/Reduce](#mapreduce), [Apache Hive](#apache-hive), [Apache Pig](#apache-pig), [Apache Spark](#apache-spark) and [Apache Storm](#apache-storm).

- [OpenSearch Hadoop](#opensearch-hadoop)
  - [Requirements](#requirements)
  - [Usage](#usage)
    - [Configuration Properties](#configuration-properties)
    - [Required](#required)
    - [Essential](#essential)
  - [Map/Reduce](#mapreduce)
    - ['Old' (`org.apache.hadoop.mapred`) API](#old-orgapachehadoopmapred-api)
    - [Reading](#reading)
    - [Writing](#writing)
    - ['New' (`org.apache.hadoop.mapreduce`) API](#new-orgapachehadoopmapreduce-api)
    - [Reading](#reading-1)
    - [Writing](#writing-1)
  - [Apache Hive](#apache-hive)
    - [Reading](#reading-2)
    - [Writing](#writing-2)
  - [Apache Pig](#apache-pig)
    - [Reading](#reading-3)
    - [Writing](#writing-3)
  - [Apache Spark](#apache-spark)
    - [Scala](#scala)
    - [Reading](#reading-4)
      - [Spark SQL](#spark-sql)
    - [Writing](#writing-4)
      - [Spark SQL](#spark-sql-1)
    - [Java](#java)
    - [Reading](#reading-5)
      - [Spark SQL](#spark-sql-2)
    - [Writing](#writing-5)
      - [Spark SQL](#spark-sql-3)
  - [Apache Storm](#apache-storm)
    - [Reading](#reading-6)
    - [Writing](#writing-6)
  - [Building the source](#building-the-source)
  - [License](#license)

## Requirements
OpenSearch (__3.x__ or higher) cluster accessible through [REST][]. That's it!
Significant effort has been invested to create a small, dependency-free, self-contained jar that can be downloaded and put to use without any dependencies. Simply make it available to your job classpath and you're set.

## Usage

### Configuration Properties

All configuration properties start with `opensearch` prefix. Note that the `opensearch.internal` namespace is reserved for the library internal use and should _not_ be used by the user at any point.
The properties are read mainly from the Hadoop configuration but the user can specify (some of) them directly depending on the library used.

### Required
```
opensearch.resource=<ES resource location, relative to the host/port specified above>
```
### Essential
```
opensearch.query=<uri or query dsl query>      # defaults to {"query":{"match_all":{}}}
opensearch.nodes=<OpenSearch host address>     # defaults to localhost
opensearch.port=<OPENSEARCH REST port>         # defaults to 9200
```

## [Map/Reduce][]

For basic, low-level or performance-sensitive environments, OpenSearch-Hadoop provides dedicated `InputFormat` and `OutputFormat` that read and write data to OpenSearch. To use them, add the `opensearch-hadoop` jar to your job classpath
(either by bundling the library along - it's ~300kB and there are no-dependencies), using the [DistributedCache][] or by provisioning the cluster manually.

Note that os-hadoop supports both the so-called 'old' and the 'new' API through its `OpenSearchInputFormat` and `OpenSearchOutputFormat` classes.

### 'Old' (`org.apache.hadoop.mapred`) API

### Reading
To read data from Os, configure the `OpenSearchInputFormat` on your job configuration along with the relevant [properties](#configuration-properties):
```java
JobConf conf = new JobConf();
conf.setInputFormat(OpenSearchInputFormat.class);
conf.set("opensearch.resource", "radio/artists");
conf.set("opensearch.query", "?q=me*");             // replace this with the relevant query
...
JobClient.runJob(conf);
```
### Writing
Same configuration template can be used for writing but using `OpenSearchOuputFormat`:
```java
JobConf conf = new JobConf();
conf.setOutputFormat(OpenSearchOutputFormat.class);
conf.set("opensearch.resource", "radio/artists"); // index or indices used for storing data
...
JobClient.runJob(conf);
```
### 'New' (`org.apache.hadoop.mapreduce`) API

### Reading
```java
Configuration conf = new Configuration();
conf.set("opensearch.resource", "radio/artists");
conf.set("opensearch.query", "?q=me*");             // replace this with the relevant query
Job job = new Job(conf)
job.setInputFormatClass(EsInputFormat.class);
...
job.waitForCompletion(true);
```
### Writing
```java
Configuration conf = new Configuration();
conf.set("opensearch.resource", "radio/artists"); // index or indices used for storing data
Job job = new Job(conf)
job.setOutputFormatClass(EsOutputFormat.class);
...
job.waitForCompletion(true);
```

## [Apache Hive][]
ES-Hadoop provides a Hive storage handler for Elasticsearch, meaning one can define an [external table][] on top of ES.

Add es-hadoop-<version>.jar to `hive.aux.jars.path` or register it manually in your Hive script (recommended):
```
ADD JAR /path_to_jar/es-hadoop-<version>.jar;
```
### Reading
To read data from ES, define a table backed by the desired index:
```SQL
CREATE EXTERNAL TABLE artists (
    id      BIGINT,
    name    STRING,
    links   STRUCT<url:STRING, picture:STRING>)
STORED BY 'org.opensearch.hive.hadoop.EsStorageHandler'
TBLPROPERTIES('opensearch.resource' = 'radio/artists', 'opensearch.query' = '?q=me*');
```
The fields defined in the table are mapped to the JSON when communicating with Elasticsearch. Notice the use of `TBLPROPERTIES` to define the location, that is the query used for reading from this table.

Once defined, the table can be used just like any other:
```SQL
SELECT * FROM artists;
```

### Writing
To write data, a similar definition is used but with a different `opensearch.resource`:
```SQL
CREATE EXTERNAL TABLE artists (
    id      BIGINT,
    name    STRING,
    links   STRUCT<url:STRING, picture:STRING>)
STORED BY 'org.opensearch.hive.hadoop.EsStorageHandler'
TBLPROPERTIES('opensearch.resource' = 'radio/artists');
```

Any data passed to the table is then passed down to Elasticsearch; for example considering a table `s`, mapped to a TSV/CSV file, one can index it to Elasticsearch like this:
```SQL
INSERT OVERWRITE TABLE artists
    SELECT NULL, s.name, named_struct('url', s.url, 'picture', s.picture) FROM source s;
```

As one can note, currently the reading and writing are treated separately but we're working on unifying the two and automatically translating [HiveQL][] to Elasticsearch queries.

## [Apache Pig][]
ES-Hadoop provides both read and write functions for Pig so you can access Elasticsearch from Pig scripts.

Register ES-Hadoop jar into your script or add it to your Pig classpath:
```
REGISTER /path_to_jar/es-hadoop-<version>.jar;
```
Additionally one can define an alias to save some chars:
```
%define ESSTORAGE org.opensearch.pig.hadoop.EsStorage()
```
and use `$ESSTORAGE` for storage definition.

### Reading
To read data from ES, use `OpenSearchStorage` and specify the query through the `LOAD` function:
```SQL
A = LOAD 'radio/artists' USING org.opensearch.pig.hadoop.EsStorage('opensearch.query=?q=me*');
DUMP A;
```

### Writing
Use the same `Storage` to write data to Elasticsearch:
```SQL
A = LOAD 'src/artists.dat' USING PigStorage() AS (id:long, name, url:chararray, picture: chararray);
B = FOREACH A GENERATE name, TOTUPLE(url, picture) AS links;
STORE B INTO 'radio/artists' USING org.opensearch.pig.hadoop.EsStorage();
```
## [Apache Spark][]
ES-Hadoop provides native (Java and Scala) integration with Spark: for reading a dedicated `RDD` and for writing, methods that work on any `RDD`. Spark SQL is also supported

### Scala

### Reading
To read data from ES, create a dedicated `RDD` and specify the query as an argument:

```scala
import org.elasticsearch.spark._

..
val conf = ...
val sc = new SparkContext(conf)
sc.esRDD("radio/artists", "?q=me*")
```

#### Spark SQL
```scala
import org.elasticsearch.spark.sql._

// DataFrame schema automatically inferred
val df = sqlContext.read.format("es").load("buckethead/albums")

// operations get pushed down and translated at runtime to Elasticsearch QueryDSL
val playlist = df.filter(df("category").equalTo("pikes").and(df("year").geq(2016)))
```

### Writing
Import the `org.elasticsearch.spark._` package to gain `savetoEs` methods on your `RDD`s:

```scala
import org.elasticsearch.spark._

val conf = ...
val sc = new SparkContext(conf)

val numbers = Map("one" -> 1, "two" -> 2, "three" -> 3)
val airports = Map("OTP" -> "Otopeni", "SFO" -> "San Fran")

sc.makeRDD(Seq(numbers, airports)).saveToEs("spark/docs")
```

#### Spark SQL

```scala
import org.elasticsearch.spark.sql._

val df = sqlContext.read.json("examples/people.json")
df.saveToEs("spark/people")
```

### Java

In a Java environment, use the `org.elasticsearch.spark.rdd.java.api` package, in particular the `JavaOpenSearchSpark` class.

### Reading
To read data from ES, create a dedicated `RDD` and specify the query as an argument.

```java
import org.apache.spark.api.java.JavaSparkContext;
import org.elasticsearch.spark.rdd.api.java.JavaOpenSearchSpark;

SparkConf conf = ...
JavaSparkContext jsc = new JavaSparkContext(conf);

JavaPairRDD<String, Map<String, Object>> esRDD = JavaOpenSearchSpark.esRDD(jsc, "radio/artists");
```

#### Spark SQL

```java
SQLContext sql = new SQLContext(sc);
DataFrame df = sql.read().format("es").load("buckethead/albums");
DataFrame playlist = df.filter(df.col("category").equalTo("pikes").and(df.col("year").geq(2016)))
```

### Writing

Use `JavaOpenSearchSpark` to index any `RDD` to Elasticsearch:
```java
import org.elasticsearch.spark.rdd.api.java.JavaOpenSearchSpark;

SparkConf conf = ...
JavaSparkContext jsc = new JavaSparkContext(conf);

Map<String, ?> numbers = ImmutableMap.of("one", 1, "two", 2);
Map<String, ?> airports = ImmutableMap.of("OTP", "Otopeni", "SFO", "San Fran");

JavaRDD<Map<String, ?>> javaRDD = jsc.parallelize(ImmutableList.of(numbers, airports));
JavaOpenSearchSpark.saveToEs(javaRDD, "spark/docs");
```

#### Spark SQL

```java
import org.opensearch.spark.sql.api.java.JavaOpenSearchSparkSQL;

DataFrame df = sqlContext.read.json("examples/people.json")
JavaOpenSearchSparkSQL.saveToEs(df, "spark/docs")
```

## [Apache Storm][]
ES-Hadoop provides native integration with Storm: for reading a dedicated `Spout` and for writing a specialized `Bolt`

### Reading
To read data from ES, use `EsSpout`:
```java
import org.opensearch.storm.OpenSearchSpout;

TopologyBuilder builder = new TopologyBuilder();
builder.setSpout("opensearch-spout", new OpenSearchSpout("storm/docs", "?q=me*"), 5);
builder.setBolt("bolt", new PrinterBolt()).shuffleGrouping("es-spout");
```

### Writing
To index data to ES, use `EsBolt`:

```java
import org.opensearch.storm.OpenSearchBolt;

TopologyBuilder builder = new TopologyBuilder();
builder.setSpout("spout", new RandomSentenceSpout(), 10);
builder.setBolt("es-bolt", new EsBolt("storm/docs"), 5).shuffleGrouping("spout");
```

## Building the source

Elasticsearch Hadoop uses [Gradle][] for its build system and it is not required to have it installed on your machine. By default (`gradlew`), it automatically builds the package and runs the unit tests. For integration testing, use the `integrationTests` task.
See `gradlew tasks` for more information.

To create a distributable zip, run `gradlew distZip` from the command line; once completed you will find the jar in `build/libs`.

To build the project, JVM 8 (Oracle one is recommended) or higher is required.

## License
This project is released under version 2.0 of the [Apache License][]

```
Licensed to Elasticsearch under one or more contributor
license agreements. See the NOTICE file distributed with
this work for additional information regarding copyright
ownership. Elasticsearch licenses this file to you under
the Apache License, Version 2.0 (the "License"); you may
not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
```

[Hadoop]: http://hadoop.apache.org
[Map/Reduce]: http://hadoop.apache.org/docs/r1.2.1/mapred_tutorial.html
[Apache Pig]: http://pig.apache.org
[Apache Hive]: http://hive.apache.org
[Apache Spark]: http://spark.apache.org
[Apache Storm]: http://storm.apache.org
[HiveQL]: http://cwiki.apache.org/confluence/display/Hive/LanguageManual
[external table]: http://cwiki.apache.org/Hive/external-tables.html
[Apache License]: http://www.apache.org/licenses/LICENSE-2.0
[Gradle]: http://www.gradle.org/
[REST]: http://www.elastic.co/guide/en/elasticsearch/reference/current/api-conventions.html
[DistributedCache]: http://hadoop.apache.org/docs/stable/api/org/apache/hadoop/filecache/DistributedCache.html