---
layout: post
title: Manual management of executors count in Spark 1.6 
date: 2017-06-29 00:00:00 +00:00
tags: [spark, executors]
---

## Dynamic allocation

Spark provides possibility to use dynamic allocation of executors. To enable it `spark.dynamicAllocation.enabled` should be set to true (there are more params that can be used to configure dynamic allocation behaviour [documentation](https://spark.apache.org/docs/1.6.1/configuration.html#dynamic-allocation)). 
However, we faced with issue that dynamic allocation doesn't work properly for spark streaming jobs. Executors are not freed up. 

> spark.dynamicAllocation.executorIdleTimeout: If dynamic allocation is enabled and an executor has been idle for more than this duration, the executor will be removed [more](https://spark.apache.org/docs/1.6.1/job-scheduling.html#resource-allocation-policy)

So, spark removes executor only if it doesn't receive any task for a while. As there is always a chance that a task from micro-batch will be assigned to executor, it will be unlikely removed. 

## Manual allocation

The job we tried to implement contained 2 parts:
1. initial load. It required much more executors than second (streaming) part
2. streaming from kafka topic (64 partitions). It required several executors to be able to process all incoming data.

Initial load time depends on number of executors. So, the more executors are used, the faster inital load will be. We tried to use dynamic allocation. Unfortunately, executors were not freed up after initial load. As there were 64 tasks (number of partitions), there were no chance to decrease executors count to 2.

Solution: request additional executors count for inital load and then remove them. 

> WARNING: based on developer API 

#### request new executors

Request new executors API is available on sparkContext. 
{% highlight scala %}
val requestResult = sparkContext.requestExecutors(num)
{% endhighlight %}

returns true if request of received successfully. 

#### delete executors

Not straightforward way to get executor ids that are used to invoke killExecutors later. 

{% highlight scala %}
val allExecutorIds = sc
         .getExecutorStorageStatus.sortBy(status => -status.numBlocks)
         .map(status => status.blockManagerId.executorId)
         .filter(_ != "driver")
{% endhighlight %}

Request to kill executor is available on spark context

{% highlight scala %}
val killResult = sc.killExecutors(executorsIdsToRemove)
{% endhighlight %}
