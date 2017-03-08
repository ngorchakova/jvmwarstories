---
layout: post
title: maxRatePerPartition with compacted topic
date: 2016-12-15 00:00:00 +00:00
tags: "spark streaming kafka"
---
I did simple spark streaming job that should read from compacted topic. As the topic was quite big I specify **'spark.streaming.kafka.maxRatePerPartition'** param in spark-submit command.

As result I got exception:

 ```
ERROR [Executor task launch worker-2] executor.Executor: Exception in task 1.0 in stage 2.0 (TID 22)
java.lang.AssertionError: assertion failed: Got 3740923 > ending offset 2428156 for topic COMPACTED.KAFKA.TOPIC partition 6 start 2228156. This should not happen, and indicates a message may have been skipped
  at scala.Predef$.assert(Predef.scala:179)
  at org.apache.spark.streaming.kafka.KafkaRDD$KafkaRDDIterator.getNext(KafkaRDD.scala:217)
 ```

Exception is not obvious. However, the problem is connected with the fact that topic was compacted. So, spark code takes the earliest offset for topicPartition and add to it maxRatePerPartition. As offsets in compacted topic are sparse, the (maxRatePerPartition)th message has offset bigger than earliestOffset + maxRatePerPartition.
