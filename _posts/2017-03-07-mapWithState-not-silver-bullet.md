---
layout: post
title: MapWithState is not a silver bullet
date: 2017-03-20 00:00:00 +00:00
tags: [spark, streaming]
---

### MapWithState features
When method `mapWithState` was announced in spark 1.6.0 ([jira ticket)](https://issues.apache.org/jira/browse/SPARK-2629), I expected that it will be heavily used everywhere.

Main difference between `mapWithState` and `updateStateByKey` is how RDD updated on each micro batch. In case of `updateStateByKey`, your update function will be invoked for ALL keys exist in RDD, even if there is nothing for them in micro batch. As result empty batch takes the same time to process as batch with some data. In case of `mapWithState`, update function will be invoked for keys that are present in micro batch. So, for empty batch there is no invocation of update function.


There is one more case when `mapWithState` function can be invoked. When `StateSpec` is configured, there is possibility to configure timeout: `def timeout(idleDuration: Duration): this.type`. If during timeout period, there is no updates for some key, then mapWithState function will be invoked for state with flag isTimingOut set to true.

{% highlight scala %}
stream
    .mapWithState(StateSpec.function((key: String, value: Option[Long], state: State[Long]) => {
    if (state.isTimingOut()) 
        println(s"key $key is going to be removed")
    ...
    }).timeout(Milliseconds(1000)))
{% endhighlight %}

Also, `mapWithState` allows to specify initial state. 

{% highlight scala %}
stream
    .mapWithState(StateSpec.function((key: String, value: Option[Long], state: State[Long]) => {
        ...
    })
    .initialState(initRdd)
    .timeout(Milliseconds(1000)))
{% endhighlight %}

### Usage: active users
There is stream with user activity events. The result should be stream of active users. User that doesn't have any events for 10 minutes should be treated as inactive. `MapWithState` perfectly suits requirements
{% highlight scala %}
stream
    .mapWithState(StateSpec.function((userId: String, value: Option[Long], state: State[Long]) => {
        if (state.isTimingOut()) {
            println(s"userId $userId is going to be removed")
        } else {
            value.foreach(state.update)
        }
        userId
    })
    .initialState(initRdd)
    .timeout(Minutes(10))))
    
{% endhighlight %}

Init rdd is provided to stream. So when job starts it already has information about active clients. For example, there is client that has last activity at `now-9m`, so it is included in initRdd. However, this client will be expired only at `now+10m`. There is [open ticket](https://issues.apache.org/jira/browse/SPARK-18563) in spark jira to add possibility to specify timeout per each row in initRdd. 

Small remark: after state was marked as timeout it is not immediately deleted. It is marked for deletion and happens only during next checkpoint. See more on [stackoverflow answer](http://stackoverflow.com/questions/36552803/spark-streaming-mapwithstate-timeout-delayed/37029378).


### Window calculations
Again: there is stream with user activity events. We want to calculate how many events user did for last 30 mins / 30 days. For 30 mins we can create dStream with window 30 minuntes and sliding - 10 seconds. However, it will not work for 30 days (of course depends on amount of data and your cluster). 

Can `mapWithState` solve that problem? Unfortunately no. In case we store list of events per client, the state recalculation will be triggered when new event for client is coming. But recalculation also should be triggered when old events are timedout. 
For example:
* at time momoment `T` client status - `[T-29d, T-5d]`. New event for client was received, the client status become `[T-29d, T-5d, T]`. The result client acitivity = 3
* at time momment `T+1` client status - `[T-30d, T-6d, T-1]`. No new event for client was received. The result client activity wasn't recalculated and is still 3. However, `T-30d` should be expired and client activity should be = 2.

`UpdateStateByKey` will solve that problem. As client status recalculation will be triggered even if there is no incoming event for client. However, overhead is huge in case of big amountof keys




