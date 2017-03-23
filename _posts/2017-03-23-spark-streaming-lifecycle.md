---
layout: post
title: Spark streaming lifecycle management
date: 2017-03-23 00:00:00 +00:00
tags: [spark, streaming, lifecycle]
---

Most of spark streaming jobs are expected to be long running jobs. However, we faced with some cases when streaming job should be stopped, then some action done and then streaming job should continue it work. 

Idea and implementation by [Dariusz Szablinski](https://www.linkedin.com/in/daroo/).
To do that, we need to manage spark streaming job lifecycle: start, stop and start again. Below you can find example of that code.

In the example the action that should be triggered periodically is called `compaction` and it is run every `compactionFrequency` seconds. 

{% highlight scala %}
  private val scheduledExecutor = Executors.newScheduledThreadPool(1)
  private val lock = new ReentrantLock()
  private val stopAndTriggerCompaction = lock.newCondition()
  private val compactionFrequency = jobConfig.jdbcCompactionFrequencySec

  def main(args: Array[String]): Unit = {
    
    val sc = createSparkContext()

    scheduledExecutor.scheduleWithFixedDelay(new Runnable {
      override def run(): Unit = {
        lock.lock()
        try {
          logger.info("Triggering compaction")
          stopAndTriggerCompaction.signalAll()
        } finally {
          lock.unlock()
        }
      }
    }, compactionFrequency, compactionFrequency, SECONDS)

    try {
      while (!currentThread().isInterrupted) {
        lock.lock()
        try {
          ... do action before streaming starts ...
          logger.info("Creating new Spark Streaming Context")
          val ssc = new StreamingContext(sc, Seconds(jobConfig.batchDuration))
          ... do preparation for streaming ...
          ssc.start()
          logger.info("Streaming has been started")
          try {
            stopAndTriggerCompaction.await()
          } finally {
            logger.info("Stopping streaming context")
            ssc.stop(stopSparkContext = false, stopGracefully = true)
            logger.info("Streaming context stopped")
          }
        } finally {
          lock.unlock()
        }
      }
    } finally {
      scheduledExecutor.shutdownNow()
      sc.stop()
    }

  }
    
{% endhighlight %}

At start up we start  separate thread (it is happening on driver). It will notify main thread that time elapsed. Meanwhile, main thread starts spark streaming context and wait for notification (`stopAndTriggerCompaction.await()`). When time passed, `ssc.stop(stopSparkContext = false, stopGracefully = true)` stops streaming context. Notice, that spark context itself is not stopped. 

