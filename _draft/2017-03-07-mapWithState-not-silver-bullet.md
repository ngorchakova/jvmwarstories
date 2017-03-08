---
layout: post
title: MapWithState is not a silver bullet
date: 2017-03-07 00:00:00 +00:00
tags: [spark, streaming]
---
When method `mapWithState` was announced in spark 1.6.0 ([jira ticket)](https://issues.apache.org/jira/browse/SPARK-2629), I expected that it will be heavily used everywhere. However, not it still has some limitations. For example, you cannot implement window calculation.

For example you have to calculate how many times user logged in during last 30 days. Flow: 
[open ticket](https://issues.apache.org/jira/browse/SPARK-18563)


{% highlight scala %}
 stream
    .map(message => (message._2, 1))
    .mapWithState(StateSpec.function((key: String, value: Option[Int], state: State[Long]) => {
        val sum = value.getOrElse(0).toLong + state.getOption.getOrElse(0L)
        val output = (key, sum)
        state.update(sum)
        println(s"MapWithState: key=$key value=$value state=$state")
        output
    }))
     .foreachRDD(rdd => {
        rdd.foreach(pair => println(s"MapWithState: key=${pair._1} value=${pair._2}"))
    })
{% endhighlight %}

