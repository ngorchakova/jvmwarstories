---
layout: post
title: Backpressure for spark streaming
date: 2016-12-23 00:00:00 +00:00
tags: [kafka, spark, streaming]
---
There is very useful feature for spark streaming jobs: back pressure. To enable it the only one parameter should be set. 

{% highlight java %}
conf.set("spark.streaming.backpressure.enabled","true")
{% endhighlight %}

General idea of back pressure: dynamically change max rate based on previous processed batch (amount of processed messages and processed time). It tries to keep processed time less than batch interval.

Except "spark.streaming.backpressure.enabled" param, there are also 4 useful that can be used to tune backpressure algorithm:

{% highlight scala %}
val proportional = conf.getDouble("spark.streaming.backpressure.pid.proportional", 1.0)
val integral = conf.getDouble("spark.streaming.backpressure.pid.integral", 0.2)
val derived = conf.getDouble("spark.streaming.backpressure.pid.derived", 0.0)
val minRate = conf.getDouble("spark.streaming.backpressure.pid.minRate", 100)
{% endhighlight %}

Based on these params new PIDRateEstimator ([**sourceCode**]((https://github.com/apache/spark/blob/master/streaming/src/main/scala/org/apache/spark/streaming/scheduler/rate/PIDRateEstimator.scala))) is created. It should be possible in the future to use another estimator (param "spark.streaming.backpressure.rateEstimator").

### Story about spark 1.6, kafka 0.9

We were using back pressure in many spark jobs. Spark UI was showing nice graphics. However, there was hidden problem: after comparing lastOffset in kafka and last processed offset in spark job we found out huge lag.
The problem was connected with using back pressure. DirectKafkaInputDStream was created for multiply topics: several hava small amount of incoming data, another has 100 times more. However, rate was calculated for all of them together and then was evenly split for all topicPartitions.

Solution: create DirectKafkaInputDStream per topic (per group of topics with the same expected rate of messages).