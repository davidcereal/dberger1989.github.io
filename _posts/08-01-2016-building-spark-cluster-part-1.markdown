---
title: "Building a Spark Cluster Part 1: Programming in Scala and Spark"
subtitle:
layout: post
date: 2016-08-01 22:48
image: /assets/images/markdown.jpg
headerImage: true
tag:
- distributed computing
- spark
- scala
- hadoop
- big data
- raspberry pi
blog: true
author: davidberger
description:   
---
## Introduction

Spark has become increasingly ubiquitous in the world of big data and is rapidly being deployed. There’s something about in-memory cluster computing that just sounds awesome. As a data scientist, I was particularly enticed by Spark’s production-ready machine learning library and streaming application extensions, and I eagerly began reading up on its theory and API. This blog post will discuss programming in Scala and Spark, and in my next post, I’ll walk through creating a Spark cluster, packaging/submitting applications, and tuning jobs for optimal performance. 

In a [previous post](https://dberger1989.github.io/tuning-hadoop-raspberry-pi/), I walked through my experience building a Hadoop cluster using four Raspberry Pi 3 computers (pictured below). I used the same four Raspberry Pis to build my Spark cluster. Setting up the Hadoop cluster was an incredible learning endeavor, and getting hands-on experience with important facets of distributed computing such as network bottlenecking and tuning container configuration made learning Spark much easier. Also,[Learning Spark](https://www.amazon.com/Learning-Spark-Lightning-Fast-Data-Analysis/dp/1449358624) by Holden Karau, Andy Konwinski, Patrick Wendell, and Matei Zaharia is very well written and helped me a lot. 

<img src ="/assets/images/post_images/picluster.jpeg" style="width:560px"/>

## Spark vs Hadoop

I’ve often heard people talk about Spark overtaking Hadoop, and while this is true insofar as Spark’s performance and functionality beat out Hadoop’s on some areas where they overlap such as batch processing, there are other areas where they don’t overlap, and are actually expected to work together. Spark does not include it’s own file system, and to distribute a resilient file system across multiple machines, Hadoop’s HDFS is still a very popular choice. Spark is built to easily work on top of HDFS, not against it. Furthermore, deploying Spark applications using Hadoop’s application manager, YARN, offers some benefits not reaped by deploying Spark using it’s own application manager, a topic which we'll touch upon later. 

## Programming in Scala

Many data scientists run Spark applications using PySpark, Spark’s Python API. Since I use Python on a daily basis, doing so would have been convenient, but I decided to take on the challenge of learning Scala, which is Spark’s native language. Many have ruminated on the pros and cons of choosing one over the other, but I don’t think anybody would argue that knowing both is an advantage. 

It only took me a couple of days to really get the hang of coding in Scala, a process that was greatly aided by coding with a Scala kernel in Jupyter’s ipython notebook using Apache Toree. As an added bonus, the Toree kernel also implements Spark, meaning you can code in the notebook as if you were using spark-shell. It’s awesome. 


## Programming in Spark-Shell

Spark’s core functionality really boils down to programming applications and submitting them for processing in a cluster. Spark-shell allows the user to program interactively in Spark, whereas in production you would build and submit applications.

At the heart of every Spark application is a SparkContext instance, which takes the form of the value `sc`. This instance is the conduit between Spark and the cluster:

```scala 
sc
output: org.apache.spark.SparkContext = org.apache.spark.SparkContext@1b7c97f
```

Using the SparkContext, we can build resilient distributed datasets, or RDDs, which are spark’s fundamental data structure: 

```scala
val lines = sc.textFile("README.md")//create an RDD caled lines
```

## Lazy Evaluation

While we have defined a value `lines` as being read from a textfile `README.md`, we have not actually stored that file in memory. Right now, `lines` is just a pointer. It’s a value with an instruction attached. Instead of interpreting lines as a variable containing the contents of `README.md`, it would be more correct to think of it as a variable that points to a set of instructions, in this case about loading a file, `README.md`. This is because when we define `lines`, Spark doesn’t have to actually produce anything. When we tell spark how to do something, but we don’t actually ask it to produce it, we call that a *transformation*. If we wanted to split each line of that `README` into words, we could tell Spark how to do it with another transformation: 
`val words = input.flatMap(line => line.split(" "))`

But, Spark still won’t have even read the `README.md` file, let alone split it up into words! Spark will only do those things when we ask it to produce something, which is known as performing an *action*. An example of an action is `collect()`, which will return the full record of an RDD. If we call `collect()` on `words`, spark performs all the necessary transformations linking up to the words value: it reads the file `README` and then splits the lines into words, and then it will return the result. `collect()` is actually used pretty infrequently in production because it requires the data to fit in the driver machine’s memory in order to return it, and if we’re working with big data, that will often be very taxing/not possible. A much more common and economic method would be `take(n)` which allows you to return only the first n results. Saving to a file is also an action because it is telling spark to produce some form of output, in this case a file.

## Partitioning an RDD

In the case above we loaded data into Spark by reading it in from an external source. We could have also created an RDD data structure by calling `sc.parallelize()`. For example, to create a list of ints from 1 to 1000, we could have done `val input = sc.parallelize(List.range(1, 1000))`. However, it is much more normal to have data read in from an external source such as HDFS, since reading all the data in through parallelize will necessitate having as much memory as the file on the driver machine.

When we create an RDD, it is distributed in memory across the nodes of our cluster in the form of partitions. If the RDD is being created from data read in from HDFS, The granularity of this partition distribution is determined by the granularity of the storage blocks. Consider a text file 300MB large that is split up on HDFS into 10MB blocks. Spark will deploy one partition per block, for a total of 30 blocks:

```scala
val input = sc.textFile("hdfs://node1:9000/test-300mb-10mbBS.txt")

input.partitions.length
//output: Int = 30
```

But, if we use an input where the blocks were more coarsely split at 15mb each, we have fewer partitions:
 
```scala
val input = sc.textFile("hdfs://node1:9000/test-300-15mbBS.txt")

input.partitions.length
//output: Int = 23
``` 

If the RDD is being created locally by data we create and parallelize, we can specify how many partitions to create:

```scala
val produceClass = List(("apple", "fruit"),("carrot", "veg"),("pear", "fruit"), ("lemon", "fruit"), ("banana", "fruit"), ("lime", "fruit"))

val input = sc.parallelize(produceClass, 2)
input.partitions.length
output: Int = 2
```

The default number of partitions is as many cores are available on the cluster. One type of partitioner is the Hash partitioner, which partitions the RDD by splitting up its keys based on each key's hashcode. Partitioning is available on all key/value pair RDDs. 

It can be very useful to use the same partitioner on multiple RDDs when we are drawing from the same data multiple times, as in a join. If we partition data from 2 RDDs consisting of key/value pairs that we plan to join using the same partitioner, we can make it much more likely (though not actually guaranteed) that matching keys from both RDDS are stored on the same machine. If we hadn’t co-partitioned the RDDs, location of keys would be random, and to join on the keys, many would have to be transferred over the network from one node to another, causing unnecessary bandwidth usage and the speed of our application would suffer. When we co-partition the RDDs, Spark groups them together on the same node, eliminating this transfer overhead. Let’s use two key/value pair lists as an example, produceClass and produceColor:

```scala
// Define 2 lists of produce key/value pairs
val produceClass = List(("apple", "fruit"),("carrot", "veg"),("pear", "fruit"), ("lemon", "fruit"), ("banana", "fruit"), ("lime", "green"))
val produceColor = List(("apple", "red"),("carrot", "orange"),("pear", "green"), ("lemon", "yellow"), 
                  ("banana", "yellow"), ("lime", "green"),("lime", "green"),("lime", "green")  )

// Create a partitioner 
val part = new HashPartitioner(3)

// Create RDDS
val classRDD = sc.parallelize(produceClass).partitionBy(part)
val colorRDD = sc.parallelize(produceColor).partitionBy(part)

// Show what’s inside each partition of the RDDs
val classRDDMapped =   classRDD.mapPartitionsWithIndex{
    (index, iterator) => {
    val Li = iterator.toList
    Li.map(x => x + " ---- partition " + index).iterator
        }
    }

val colorRDDMapped =   colorRDD.mapPartitionsWithIndex{
    (index, iterator) => {
    val Li = iterator.toList
    Li.map(x => x + " ---- partition " + index).iterator
        }
    }
```
```
classRDD
(lime,fruit) ---- partition 0
(pear,fruit) ---- partition 1
(apple,fruit) ---- partition 2
(carrot,veg) ---- partition 2
(lemon,fruit) ---- partition 2
(banana,fruit) ---- partition 2

colorRDD
(lime,green) ---- partition 0
(pear,green) ---- partition 1
(banana,yellow) ---- partition 2
(apple,red) ---- partition 2
(lemon,yellow) ---- partition 2
(carrot,orange) ---- partition 2
```
As we can see, values of the same key wind up partitioned with the same partition number. Without using the same partitioner, matching keys would not be on the same partition. It would have been some variation of:

```
classRDD-no-copartition
(apple,fruit) ---- partition 0
(carrot,veg) ---- partition 0
(pear,fruit) ---- partition 1
(lemon,fruit) ---- partition 1
(banana,fruit) ---- partition 2
(lime,fruit) ---- partition 2

colorRDD-no-copartition
(banana,yellow) ---- partition 0
(lime,green) ---- partition 0
(pear,green) ---- partition 1
(apple,red) ---- partition 1
(lemon,yellow) ---- partition 2
(carrot,orange) ---- partition 2
```

This is critical, because partitions of the same number will be stored on the same machine removing the need for the expensive cross-machine shuffle:

<img src ="/assets/images/post_images/spark_cluster_1/join_noncopartitioned.svg" style="width:560px"/>

<img src ="/assets/images/post_images/spark_cluster_1/join_copartitioned.svg" style="width:560px"/>

As we can see in the 2 diagrams above, when matching keys from different RDDs are stored on the same node, it eliminates costly shuffles between nodes across the network. 

In addition to hash partitioning, there is range partitioning. This is useful if the key values are not random and won’t be split up evenly across nodes. The partition a hashed key goes to is calculated by: `key_hashcode % n_partitions`. So if keys are ordinal and there are many that end up with a hashcode ending in the number `0`, and we have 10 partitions, `key_hashcode % n_partitions` will always be `0` since there will never be a remainder. In such a case, all the keys would go into partition `0`, and many nodes/cores will go unused. If we partition based on range, we’ll avoid the hashcode problem, although it’s entirely possible that we would also get a lopsided partitioning if our key distribution was lopsided in it's distribution across that range. 


## Persisting an RDD

Since Spark lazily evaluates objects, calling multiple objects on the same RDD can result in the same loading and transformation lineage being done multiple times. Consider this word count example where we not only count how many times each word appears, but we also count the total number of words in the document:

``` scala
val input = sc.textFile("README.md")
val words = input.flatMap(x => x.split(" “))

val wordCount = words.map(x => (x, 1)).reduceByKey((x, y) => x + y)
wordCount.take(5)

val totalCounted = words.count()
totalCounted.take(5)
```

When we call the `take()` action on `wordCount` and `totalCounted`, we are both times necessitating that the transformation lineage to be executed. This means that `input` and `words` will have to be evaluated twice! 

To avoid this problem, we can use `persist()` to store (cache) `words` in memory, allowing it to be used by multiple actions without having to be recomputed:

``` scala
val input = sc.textFile("README.md")
val words = input.flatMap(x => x.split(" “))

// Create persistence
// MEMORY_AND_DISK allows for spilling to disk when there is too little memory
persist(StorageLevel.MEMORY_AND_DISK) 

val wordCount = words.map(x => (x, 1)).reduceByKey((x, y) => x + y)
wordCount.take(5)

val totalCounted = words.count()
totalCounted.take(5)
```

## Broadcast Variables

The last data structure i’ll talk about is Broadcast Variables. Consider a situation we have text for which we’d like to count how many times each grammatical element of speech occurs. Each time we classify a word, we have to look up it’s element in a table, `wordType`. If we cached this mapping, it would be loaded into memory on one node, but on the other nodes, each word lookup has to be done over the network. This is because `wordType` is not stored locally on each node. To do this and solve the network overhead problem, we create a Broadcast variable, which means Spark broadcasts that RDD to each node and caches it:

```scala
// Create map of word to type of speech 
val wordTypeMap = Map("jump" -> "verb", "over" -> "preposition", "box" -> "noun")

// Make wordTypeMap a broadcast variable 
val wordTypeMapBroadcast = sc.broadcast(wordType)

val sent = sc.parallelize(List("jump over box", "jump box jump jump"))
val words = sent.flatMap(x => x.split(" "))

// Replace each word with it's word type and count word types
val wordTypeCounts = words.map(x => (wordTypeMapBroadcast.value(x), 1)).reduceByKey((x, y) => x + y)

wordTypeCounts.collect()

//output: Array((preposition,1), (noun,2), (verb,4))
```


## Stay tuned
In the walkthrough above, we engaged in various topics important to programming in Spark. In my next post, I'll detail the steps I took to turn my Hadoop Raspberry Pi cluster into a Hadoop+Spark cluster, as well as some of the intricacies of submitting a job in Spark. Perhaps most importantly, I'll also discuss the various ways you can ensure your job juns at optimal efficiency by tuning Spark's config parameters.






