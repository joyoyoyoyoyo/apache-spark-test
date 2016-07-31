# Apache Spark Test
A small study project on how to create and run applications with [Apache Spark][spark]. Its my personal study project and is mostly a copy/paste of a lot of resources available on the Internet to get the concepts on one page.

# Introduction
Apache Spark is an Open Source distributed general-purpose cluster computing framework with in-memory data processing engine that can do ETL, analytics, machine learning and graph processing on large volumes of data at rest (batch processing) or in motion (streaming processing) with rich concise high-level APIs for the programming languages: Scala, Python, Java, R, and SQL.

You could also describe Spark as a distributed, data processing engine for batch and streaming modes featuring SQL queries, graph processing, and Machine Learning.

Spark is often called cluster computing engine or simply execution engine.

In contrast to Hadoop’s two-stage disk-based MapReduce processing engine, Spark’s multi-stage in-memory computing engine allows for running most computations in memory, and hence very often provides better performance (there are reports about being up to 100 times faster for certain applications, e.g. iterative algorithms or interactive data mining.

# Downloading Apache Spark
[Downloaded Spark][spark-download], since we won’t be using HDFS, you can download a package for any version of Hadoop.

# Stand-alone Spark cluster
For this project, I am only interested in the stand-alone Spark cluster mode, which consists of a single master node, which is a single point of failure, and multiple worker nodes.

## Launching master
There can only be a single master in [standalone-mode][standalone-mode]. The standalone master can be started by executing:

```bash
$SPARK_HOME/sbin/start-master.sh -h localhost -p 7077
```

Once started, the master will print out a `spark://HOST:PORT` URL for itself, which you can use to connect workers to it, or pass as the _master_ argument to SparkContext. You can also find this URL on the master’s web UI, which is [http://localhost:8080](http://localhost:8080) by default.

Please replace `localhost` by the network ip address if you wish to connect remote workers to this master and configure the remote workers to use the ip address of the master.

## Launching worker
Workers (slaves) connect to the master. Once you have started a worker that has been connected to a master, look at the master’s web UI [http://localhost:8080](http://localhost:8080). You should see the new node listed there, along with its number of _CPUs_ and _memory_.

To start and connect a worker, execute the following:

```bash
$SPARK_HOME/sbin/start-slave.sh spark://localhost:7077 -c 1 -m 1G
```

This will launch a single worker that will connect to your local master at port 7077.

To stop the worker, execute the following command:

```bash
$SPARK_HOME/sbin/stop-slave.sh
```

## Launching multiple workers
To launch multiple workers on a node, you should export the environment variable `SPARK_WORKER_INSTANCES` and then launch the script `start-slave.sh`:

```bash
export SPARK_WORKER_INSTANCES=4
$SPARK_HOME/sbin/start-slave.sh spark://localhost:7077 -c 1 -m 1G
```

## Stopping workers
To stop the workers, execute the following:

```bash
export SPARK_WORKER_INSTANCES=4
$SPARK_HOME/sbin/stop-slave.sh
```

The script will use the `SPARK_WORKER_INSTANCES` if found, and stop all the workers.

# Hands-on with Spark

## Launching the interactive shell
Spark can run interactively through a modified version of the Scala shell (REPL). This is a great way to learn the API. To launch the shell execute the following command, which will connect to the master running on `localhost`:

```bash
$SPARK_HOME/bin/spark-shell --master spark://localhost:7077
```

The [--master](http://spark.apache.org/docs/latest/submitting-applications.html#master-urls) option specifies the master URL for a distributed cluster, or local to run locally with one thread, or local[N] to run locally with N threads. You should start by using local for testing. For a full list of options, run Spark shell with the --help option.

## Spark Context

In the Spark shell you get an initialized [org.apache.spark.SparkContext][sparkcontext] with the value `sc`:

```scala
scala> sc
res1: org.apache.spark.SparkContext = org.apache.spark.SparkContext@5623fd90
```

You also get an instance of the [org.apache.spark.sql.SQLContext][sqlcontext] with the value `spark` to be used with [Spark SQL][sparksql]:

```scala
scala> spark
res2: org.apache.spark.sql.SparkSession = org.apache.spark.sql.SparkSession@6aa27145
```

## Counting
You should now be able to execute the following command, which should return 1000:

```scala
scala> sc.parallelize(1 to 1000).count()
res0: Long = 1000
```

## Distributed Pi Calculation
To calculate Pi, we can create the following application:

```scala
:paste
import scala.math.random
val slices = 50
val n = math.min(10000000L * slices, Int.MaxValue).toInt // avoid overflow
val count = sc.parallelize(1 until n, slices).map { i =>
  val x = random * 2 - 1
  val y = random * 2 - 1
  if (x*x + y*y < 1) 1 else 0
}.reduce(_ + _)
println("Pi is roughly " + 4.0 * count / (n - 1))

<CTRL+D>

Pi is roughly 3.1415012702830025
import scala.math.random
slices: Int = 50
n: Int = 500000000
count: Int = 392687658
```

Note: calculating Pi is faster with less slices like eg: 2.

## Processing a file
To process a file, we first need to get one. Let's download the Spark `readme.md` file and put it into `/tmp`

```bash
wget -O /tmp/readme.md https://raw.githubusercontent.com/apache/spark/master/README.md
```

Spark’s primary abstraction is a distributed collection of items called a [Resilient Distributed Dataset (RDD)][rdd].
RDDs can be created from various sources, like for example the `Range` from the _Counting_ example above. We will create
an RDD from a text file, the `/tmp/readme.md`:

```scala
scala> val textFile = sc.textFile("/tmp/readme.md")
textFile: org.apache.spark.rdd.RDD[String] = /tmp/readme.md MapPartitionsRDD[1] at textFile
```

RDDs have actions, which return values, and transformations, which return pointers to new RDDs. Let’s start with a few actions:

```scala
scala> textFile.count() // Number of items in this RDD
res0: Long = 99

scala> textFile.first() // First item in this RDD
res1: String = # Apache Spark
```

Now let’s use a transformation. We will use the filter transformation to return a new RDD with a subset of the items in the file.

```scala
scala> val linesWithSpark = textFile.filter(line => line.contains("Spark"))
linesWithSpark: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[2] at filter at <console>:26
```

We can chain together transformations and actions:

```scala
scala> textFile.filter(line => line.contains("Spark")).count() // How many lines contain "Spark"?
res2: Long = 19
```

RDD actions and transformations can be used for more complex computations. Let’s say we want to find the line with the most words:

```scala
scala> textFile.map(line => line.split(" ").size).reduce((a, b) => if (a > b) a else b) // line with most words
res3: Int = 22
```

One common data flow pattern is MapReduce, as popularized by Hadoop. Spark can implement MapReduce flows easily:

```scala
scala> val wordCounts = textFile.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey((a, b) => a + b)
wordCounts: org.apache.spark.rdd.RDD[(String, Int)] = ShuffledRDD[7] at reduceByKey at <console>:26
```

Here, we combined the flatMap, map, and reduceByKey transformations to compute the per-word counts in the file as an RDD
of (String, Int) pairs. To collect the word counts in our shell, we can use the collect action:

```scala
scala> wordCounts.collect()
res4: Array[(String, Int)] = Array((package,1), (this,1), ...)
```

## Caching
Spark also supports pulling data sets into a cluster-wide in-memory cache. This is very useful when data is accessed repeatedly,
such as when querying a small “hot” dataset or when running an iterative algorithm like PageRank. As a simple example, let’s
mark our linesWithSpark dataset to be cached:

```scala
scala> linesWithSpark.cache()
res5: linesWithSpark.type = MapPartitionsRDD[2] at filter at <console>:26

scala> linesWithSpark.count()
res5: Long = 19

scala> linesWithSpark.count()
res6: Long = 19
```

It may seem silly to use Spark to explore and cache a 100-line text file. The interesting part is that these same functions can be used on very large data sets, even when they are striped across tens or hundreds of nodes.

## Spark Web UI
The Web UI (aka webUI or Spark UI after SparkUI) is the web interface of a Spark application to inspect job executions in the SparkContext using a browser.
Every SparkContext launches its own instance of Web UI which is available at http://[ipaddress]:4040 by default (the port can be changed using spark.ui.port setting).

It has the following settings:

- __spark.ui.enabled (default: true)__: controls whether the web UI is started at all
- __spark.ui.port (default: 4040)__: the port Web UI binds to
- __spark.ui.killEnabled (default: true)__:  whether or not you can kill stages in web UI.

# Handy start an stop scripts
You can put the following three scripts in the root of your spark distribution:

## start.sh

```bash
#!/bin/bash
export SPARK_WORKER_INSTANCES=4
sbin/start-master.sh -h localhost -p 7077
sbin/start-slave.sh spark://localhost:7077 -c 1 -m 1G
```

## stop.sh

```bash
#!/bin/bash
export SPARK_WORKER_INSTANCES=4
sbin/stop-slave.sh
sbin/stop-master.sh
```

## shell.sh

```bash
#!/bin/bash
export SPARK_EXECUTOR_INSTANCES=4
bin/spark-shell --master spark://localhost:7077 --verbose
```

# Working with RDDs
Spark revolves around the concept of a [Resilient Distributed Dataset (RDD)][rdd], which is a fault-tolerant collection of elements that can be operated on in parallel. There are two ways to create RDDs: parallelizing an existing collection in your driver program, or referencing a dataset in an external storage system, such as a shared filesystem, HDFS, HBase, or any data source offering a Hadoop InputFormat.

The [Resilient Distributed Dataset (RDD)][rdd] is the primary data abstraction in Apache Spark and the core of Spark (that many often refer to as Spark Core).

A RDD is a resilient and distributed collection of records. One could compare RDD to a Scala collection (that sits on a single JVM) to its distributed variant (that sits on many JVMs, possibly on separate nodes in a cluster).

With RDD the creators of Spark managed to hide data partitioning and so distribution that in turn allowed them to design parallel computational framework with a higher-level programming interface (API).

An RDD is:
- __Resilient__: fault-tolerant with the help of RDD lineage graph (each RDD remembers how it was built from other datasets (by transformations like map, join or groupBy) to rebuild itself) and so able to recompute missing or damaged partitions due to node failures.
- __Distributed__: with data residing on multiple nodes in a cluster.
- __Dataset__: is a collection of partitioned data with primitive values or values of values, e.g. tuples or other objects (that represent records of the data you work with).

Beside the above traits (that are directly embedded in the name of the data abstraction - RDD) it has the following additional traits:

- __In-Memory__: data inside RDD is stored in memory as much (size) and long (time) as possible.
- __Immutable or Read-Only__: it does not change once created and can only be transformed using transformations to new RDDs.
- __Lazy evaluated__: the data inside RDD is not available or transformed until an action is executed that triggers the execution.
- __Cacheable__: you can hold all the data in a persistent "storage" like memory (default and the most preferred) or disk (the least preferred due to access speed).
- __Parallel__: process data in parallel.
- __Typed__: values in a RDD have types, e.g. RDD[Long] or RDD[(Int, String)].
- __Partitioned__: the data inside a RDD is partitioned (split into partitions) and then distributed across nodes in a cluster (one partition per JVM that may or may not correspond to a single node).

An RDD is a named (by name) and uniquely identified (by id) entity inside a SparkContext. It lives in a SparkContext and as a SparkContext creates a logical boundary, RDDs can’t be shared between SparkContexts. An RDD can optionally have a friendly name accessible using name that can be changed.

Lets create an RDD:

```scala
scala> val xs = sc.parallelize(0 to 10)
xs: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[19]

scala> xs.id
res0: Int = 19

scala> xs.name
res1: String = null

scala> xs.name = "my first rdd"
xs.name: String = my first rdd

scala> xs.name
res2: String = my first rdd

scala> xs.toDebugString
res3: String = (4) my first rdd ParallelCollectionRDD[19]

scala> scala> xs.count
res4: Long = 11

scala> xs.take(2).foreach(println)
0
1
```

RDDs are a container of instructions on how to materialize big (arrays of) distributed data, and how to split it into partitions so Spark (using executors) can hold some of them.

In general, data distribution can help executing processing in parallel so a task processes a chunk of data that it could eventually keep in memory.

Spark does jobs in parallel, and RDDs are split into partitions to be processed and written in parallel._Inside a partition, data is processed sequentially!_

Saving partitions results in part-files instead of one single file (unless there is a single partition).

# Types of RDDs
The following are examples of RDDs:

- org.apache.spark.rdd.JdbcRDD: an RDD that executes a SQL query on a JDBC connection and reads results.
- org.apache.spark.rdd.ParallelCollectionRDD: slice a collection into numSlices sub-collections. One extra thing we do here is to treat Range collections specially, encoding the slices as other Ranges to minimize memory cost. This makes it efficient to run Spark over RDDs representing large sets of numbers. And if the collection is an inclusive Range, we use inclusive range for the last slice.
- org.apache.spark.rdd.CoGroupedRDD: an RDD that cogroups its parents. For each key k in parent RDDs, the resulting RDD contains a tuple with the list of values for that key. Should not be instantiated directly but instead use `RDD.cogroup(...)`
- org.apache.spark.rdd.HadoopRDD: an RDD that provides core functionality for reading data stored in HDFS using the older MapReduce API. The most notable use case is the return RDD of SparkContext.textFile.
- org.apache.spark.rdd.MapPartitionsRDD: an RDD that applies the provided function to every partition of the parent RDD; a result of calling operations like map, flatMap, filter, mapPartitions, etc.
- org.apache.spark.rdd.CoalescedRDD: Represents a coalesced RDD that has fewer partitions than its parent RDD; a result of calling operations like repartition and coalesce
- org.apache.spark.rdd.ShuffledRDD: the resulting RDD from a shuffle (e.g. repartitioning of data); a result of shuffling, e.g. after repartition and coalesce
- org.apache.spark.rdd.PipedRDD: an RDD that pipes the contents of each parent partition through an external command (printing them one per line) and returns the output as a collection of strings; an RDD created by piping elements to a forked external process.
- PairRDD (implicit conversion by org.apache.spark.rdd.PairRDDFunctions): that is an RDD of key-value pairs that is a result of groupByKey and join operations.
- DoubleRDD (implicit conversion as org.apache.spark.rdd.DoubleRDDFunctions) that is an RDD of Double type.
- SequenceFileRDD (implicit conversion as org.apache.spark.rdd.SequenceFileRDDFunctions) that is an RDD that can be saved as a SequenceFile.

# Actions
Actions are RDD operations that produce non-RDD values. They materialize a value in a Spark program. In other words, an RDD operation that returns a value of any type.
Actions are synchronous. Note that you can use AsyncRDDActions to release a calling thread while calling actions. Async operations return types that all inherit from `scala.concurrent.Future[T]`.

Actions trigger execution of RDD transformations to return values. Simply put, an action evaluates the RDD lineage graph. Actions materialize the entire processing pipeline with real data.
Actions are one of two ways to send data from _executors_ to the _driver_ (the other being accumulators).

Actions run _jobs_ using `SparkContext.runJob` or directly `DAGScheduler.runJob`.

__Performance tip__: You should _cache_ RDDs you work with when you want to execute two or more actions on it for a better performance.

The following are a subset of actions that is available on `org.apache.spark.rdd.RDD` (there are a lot more):

- aggregate: aggregate the elements of each partition, and then the results for all the partitions, using given combine functions and a neutral "zero value". This function can return a different result type, U, than the type of this RDD, T. Thus, we need one operation for merging a T into an U and one operation for merging two U's, as in scala.TraversableOnce. Both of these functions are allowed to modify and return their first argument instead of creating a new U to avoid memory allocation.
- collect: Return an array that contains all of the elements in this RDD. __Note:__ this method should only be used if the resulting array is expected to be small, as all the data is loaded into the driver's memory.
- count(): Return the number of elements in the RDD.
- countApprox(timeout: Long): Approximate version of count() that returns a potentially incomplete result within a timeout, even if not all tasks have finished.
- countByValue: Return the count of each unique value in this RDD as a local map of (value, count) pairs. Note that this method should only be used if the resulting map is expected to be small, as the whole thing is loaded into the driver's memory.
- first: Return the first element in this RDD.
- fold: Aggregate the elements of each partition, and then the results for all the partitions, using a given associative function and a neutral "zero value". The function `op(t1, t2)` is allowed to modify t1 and return it as its result value to avoid object allocation; however, it should not modify t2. This behaves somewhat differently from fold operations implemented for non-distributed collections in functional languages like Scala. This fold operation may be applied to partitions individually, and then fold those results into the final result, rather than apply the fold to each element sequentially in some defined ordering. For functions that are not commutative, the result may differ from that of a fold applied to a non-distributed collection.
- foreach: Applies a function f to all elements of this RDD.
- foreachPartition: Applies a function f to each partition of this RDD.
- max: Returns the max of this RDD as defined by the implicit Ordering[T]
- min: Returns the min of this RDD as defined by the implicit Ordering[T]
- reduce: Reduces the elements of this RDD using the specified commutative and associative binary operator.
- take: Take the first num elements of the RDD. It works by first scanning one partition, and use the results from that partition to estimate the number of additional partitions needed to satisfy the limit.           * @note this method should only be used if the resulting array is expected to be small, as all the data is loaded into the driver's memory.
- takeOrdered: Returns the first k (smallest) elements from this RDD as defined by the specified implicit Ordering[T] and maintains the ordering. This does the opposite of `top`.
- takeSample: Return a fixed-size sampled subset of this RDD in an array. __Note__ this method should only be used if the resulting array is expected to be small, as all the data is loaded into the driver's memory.
- toLocalIterator: * Return an iterator that contains all of the elements in this RDD. The iterator will consume as much memory as the largest partition in this RDD. Note: this results in multiple Spark jobs, and if the input RDD is the result of a wide transformation (e.g. join with different partitioners), to avoid recomputing the input RDD should be cached first.
- top: Returns the top k (largest) elements from this RDD as defined by the specified implicit Ordering[T] and maintains the ordering. This does the opposite of `takeOrdered`.
- treeAggregate: Aggregates the elements of this RDD in a multi-level tree pattern.
- treeReduce: Reduces the elements of this RDD in a multi-level tree pattern.
- saveAsTextFile: Save this RDD as a text file, using string representations of elements.
- saveAsTextFile(CompressionCodec): Save this RDD as a compressed text file, using string representations of elements.
- saveAsObjectFile: Save this RDD as a SequenceFile of serialized objects.

# AsyncRDDActions
AsyncRDDActions are RDD operations that produce non-RDD values in an asynchronous manner. These operations releases the calling thread and return a type that inherits from `scala.concurrent.Future[T]`.

The following asynchronous methods are available:
- countAsync
- collectAsync
- takeAsync
- foreachAsync
- foreachPartitionAsync


# Extensions for Apache Spark

## XML Data Source for Apache Spark
Note: Only compatible with Spark 1.x
[Spark XML][sparkxml] is a library for parsing and querying XML data with Apache Spark, for Spark SQL and DataFrames.

## CSV Data Source for Apache Spark
Note: Only compatible with Spark 1.x
[Spark CSV][sparkcsv] is a library for parsing and querying CSV data with Apache Spark, for Spark SQL and DataFrames.

# Books
- [Jacek Laskowski - Mastering Apache Spark (Free)](https://www.gitbook.com/book/jaceklaskowski/mastering-apache-spark/)

# Papers
- [Matei Zaharia et al. - Resilient Distributed Datasets: A Fault-Tolerant Abstraction for
In-Memory Cluster Computing][rddpaper]

# Video Resources

* [Apache Spark - How to install](https://www.youtube.com/watch?v=L5QWO8QBG5c)
* [Intro to Apache Spark - A Brain Friendly Tutorial](https://www.youtube.com/watch?v=rvDpBTV89AM)

[spark]: http://spark.apache.org/
[standalone-mode]: http://spark.apache.org/docs/latest/spark-standalone.html
[spark-download]: http://spark.apache.org/downloads.html
[sparkcontext]: http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.SparkContext
[sqlcontext]: https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.SQLContext
[sparksql]: http://spark.apache.org/docs/latest/sql-programming-guide.html
[rdd]: http://spark.apache.org/docs/latest/programming-guide.html#resilient-distributed-datasets-rdds
[rddpaper]: https://people.eecs.berkeley.edu/~matei/papers/2012/nsdi_spark.pdf
[jdbcrdd]: http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.JdbcRDD

[databricks]: https://databricks.com/
[sparkxml]: https://github.com/databricks/spark-xml
[sparkcsv]: https://github.com/databricks/spark-csv