---
layout: post
title: DStream updateStateByKey vs mapWithState
date: 2017-01-13 00:00:00 +00:00
tags: [spark, streaming]
---
Complex event processing problems are based on storing some current state and updating it based on incoming events (i.e. change of state). Spark API proposes two function to do that:
* `updateStateByKey`
* `mapWithState` (since spark 1.6.0)
Both these functions:
* are executed on [PairDStream](https://spark.apache.org/docs/1.6.0/api/java/org/apache/spark/streaming/dstream/PairDStreamFunctions.html) ([JavaPairDStreaml](https://spark.apache.org/docs/1.6.0/api/java/org/apache/spark/streaming/api/java/JavaPairDStream.html))
* require checkpoints enabled
Main difference of these functions is 'for what keys state will be recalculated':	
* `updateStateByKey` is executed on the whole range of keys in DStream. As results performance of these operation is proportional to the size of the state
* `mapWithState` is executed only on set of keys that are available in the last micro batch. As result performance is proportional to the size of the batch

### Simple example of streaming app that calculates count per each message value:

{% highlight scala %}
stream
    .map(message => (message._2, 1))
    .updateStateByKey((newValues: Seq[Int], oldValue: Option[Int]) => {
        println(s"new values = ${newValues.mkString(",")} oldValue=$oldValue")
        Some(newValues.foldLeft(oldValue.getOrElse(0))(_ + _))
    })
    .foreachRDD(rdd => {
        rdd.foreach(pair => println(s"k=${pair._1} v=${pair._2}"))
    })
{% endhighlight %}

and the similar code for `mapWithState`:
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

Streaming timeline:

1. micro batch contains one message 'a'

    * UpdateStateByKey: new values = 1 oldValue=None  (for key=a. No value before)
    * UpdateStateByKey: key=a value=1
    * MapWithState: key=a value=Some(1) state=1
    * MapWithState: key=a value=1
    
2. micro batch contains messages 'a' and 'b'

    * UpdateStateByKey: new values = 1 oldValue=Some(1) (for key a. Old value is 1)
    * UpdateStateByKey: new values = 1 oldValue=None (for key b. No value before)
    * UpdateStateByKey: key=b value=1
    * UpdateStateByKey: key=a value=2
    * MapWithState: key=b value=Some(1) state=1
    * MapWithState: key=a value=Some(1) state=2
    * MapWithState: key=b value=1
    * MapWithState: key=a value=2
    
3. micro batch contains one message 'b'

    * UpdateStateByKey: new values = oldValue=Some(2)
    * UpdateStateByKey: key=a value=2
    * UpdateStateByKey: new values = 1 oldValue=Some(1)
    * UpdateStateByKey: key=b value=2
    * MapWithState: key=b value=Some(1) state=2
    * MapWithState: key=b value=2

First and the second stages are processed by both functions similarly. Both `updateStateByKey` and `mapWithState` are executed for all incoming values from one side, and for all keys stored in state from another.
The main difference is in the third stage. `UpdateStateByKey` is invoked on key 'a' and 'b', while `MapWithState` only on 'b', as there is no incoming updates for 'a'. This approach increase performance of processing state in DStream up to 8 times [databrics benchmarks](https://databricks.com/blog/2016/02/01/faster-stateful-stream-processing-in-apache-spark-streaming.html)
