---
layout: post
title: Support of timestamp type in Avro by Spark
date: 2020-05-25 00:00:00 +00:00
tags: [spark, avro, databricks, timestamp]
---
## Environment
We have a spark job (spark version 2.2.0) that reads avro files, massages them and saves them as avro and parquet files back.  
One of actions performed is: converting two fields (epoch seconds and nanos) to one field timestamp (and casting it to spark logical type TimestampType)

`convertToMillis(df.col("seconds"), df.col("nanoseconds")).cast(TimestampType)`

After this restuled avro files are uploaded to Redshift. The corresponding column in Redshift is TIMESTAMP. 


## Problem
During upgrade of spark from 2.2.0 to 2.4.4 uploading to Redshift stopped working. 

What has changed: in version 2.4.0 built-in Avro support was added to Spark [SPARK-24768](https://issues.apache.org/jira/browse/SPARK-24768) (to replace databricks spark-avro library). As result in the new version spark stores timestamp field as a long in *microseconds*, while in the old version in *milliseconds*. If the avro files are processed only by spark (old or new version), the change is invisible. However, if you use a vanila avro library to read the data, the value will be 1000 bigger than expected.   

## Example how to reproduce

Avro schema

{% highlight json %}
{
  "name": "Event",
  "namespace": "test",
  "type": "record",
  "fields": [
    {"name": "epochSeconds", "type": ["null", "long"], "default": null, "doc" : "epoch seconds"}
  ]
}
{% endhighlight %}


For spark 2.2 with the dependecy com.databricks:spark-avro_2.11:4.0.0

{% highlight scala %}
test("avro -> read avro as spark -> cast timestamp -> write via spark -> read as avro reader") {
    val epochSeconds = System.currentTimeMillis() / 1000
    val schema = getSchema
    val record = createRecordWithEpochSeconds(schema, epochSeconds)
    record.get("epochSeconds") shouldEqual epochSeconds

    val tempSourceFile = File.createTempFile("timestamp-", ".avro")
    writeRecord(schema, record, tempSourceFile)

    //read avro file by spark
    val df = spark.read.format("com.databricks.spark.avro").load(s"file://${tempSourceFile.getPath}")
    val castedDf = df.withColumn("epochSeconds_casted", df.col("epochSeconds").cast(TimestampType))

    //save casted column as an avro
    val tempCastedAvroDestination = File.createTempFile("timestamp-", ".avro")
    castedDf.write.mode(SaveMode.Overwrite).avro(s"file://${tempCastedAvroDestination.getPath}")

    //read from casted avro
    val tempCastedAvroDestinationFile = tempCastedAvroDestination.listFiles(new FilenameFilter() {
      override def accept(dir: File, name: String): Boolean = name.endsWith(".avro") && name.startsWith("part")
    }).head
    val castedRecord = readOneRecordFromAvro(tempCastedAvroDestinationFile)
    castedRecord.get("epochSeconds") shouldEqual epochSeconds
    castedRecord.get("epochSeconds_casted") shouldEqual epochSeconds*1000
}
  
private def getSchema = new Schema.Parser().parse(new File(getClass.getResource("/event.avsc").getPath))

private def createRecordWithEpochSeconds(schema: Schema, epochSeconds: Long) =
    new GenericRecordBuilder(schema).set("epochSeconds", epochSeconds).build()

{% endhighlight %}


For spark 2.4.0 only small changes in the code are required and the databricks library needs to be replaced with org.apache.spark:spark-avro_2.11:2.4.0
{% highlight scala %} 
castedDf.write.mode(SaveMode.Overwrite).format("avro").save(s"file://${tempCastedAvroDestination.getPath}")
{% endhighlight %}

After this the last assertion is failing. As the epochSeconds_casted column has a value in microseconds (not in milliseconds as before). The schema after version upgrade looks as
{% highlight json %} 
{
    "name" : "epochSeconds_casted",
    "type" : [ {
      "type" : "long",
      "logicalType" : "timestamp-micros"
    }, "null" ]
x}
{% endhighlight %}

## Workaround
We removed casting to Timestamp (in case of avro), but left casting to Timestamp in case of Parquet files. 