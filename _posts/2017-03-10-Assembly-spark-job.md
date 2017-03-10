---
layout: post
title: Assembly spark job (for 1.6)
date: 2017-03-10 00:00:00 +00:00
tags: [spark]
---

To run spark job on hadoop cluster you need to have `uber jar` with all dependencies required for your job to be run. However, some of the dependencies are provided by spark itself. It is [recomended](https://spark.apache.org/docs/1.6.2/submitting-applications.html) to mark them as `provided` in pom.xml. 

All dependencies proveded by spark can be found in spark-assembly file. There is file `DEPENDENCIES` in META-INF folder. 

part of this file:

> 
  - stream-lib (https://github.com/addthis/stream-lib) com.clearspring.analytics:stream:jar:2.7.0
    License: Apache License, Version 2.0  (http://www.apache.org/licenses/LICENSE-2.0.txt)
  - Kryo (http://code.google.com/p/kryo/) com.esotericsoftware.kryo:kryo:bundle:2.21
    License: New BSD License  (http://www.opensource.org/licenses/bsd-license.php)
  ...

In case of hortoworks distribution, path of this file is /usr/hdp/hadoop_version/spark/lib/spark-assembly-spark+hadoop_version.jar 

Additionaly you do not need to provive in your jar:
* spark-core
* spark-bagel
* spark-mllib
* spark-streaming
* spark-graphx
* spark-sql

Anything else for spark should be added to jar. For example: spark-streaming-kafka
