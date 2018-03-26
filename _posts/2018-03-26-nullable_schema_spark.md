---
layout: post
title: Nullable schema if read avro files by spark
date: 2018-03-26 00:00:00 +00:00
tags: [avro, spark]
---
## Versions
Spark - 2.2

## Problem

If you read avro files by spark you can notice that in the resulted schema all fields are optional. 

For example, if you read avro file with schema to dataframe

{% highlight json %}
{
	"type": "record",
	"name": "user",
	"namespace": "kakfa-avro.test",
	"fields": [
		{
			"name": "id",
			"type": "int"
		}
  ]
}
{% endhighlight %}

And then save dataframe to avro file back, then resulted schema will be:

{% highlight json %}
{
	"type": "record",
	"name": "user",
	"namespace": "kakfa-avro.test",
	"fields": [
		{
			"name": "id",
			"type": [
				"int",
				"null"
			]
		}
  ]
}
{% endhighlight %}

There is oen difference in the schema: the field id became optional. That is happening because of one line `dataSchema = dataSchema.asNullable` in DataSource class [source code](https://github.com/apache/spark/blob/master/sql/core/src/main/scala/org/apache/spark/sql/execution/datasources/DataSource.scala)

{% highlight scala %}
HadoopFsRelation(
          fileCatalog,
          partitionSchema = partitionSchema,
          dataSchema = dataSchema.asNullable,
          bucketSpec = bucketSpec,
          format,
          caseInsensitiveOptions)(sparkSession)
{% endhighlight %}

There is a [jira ticket](https://issues.apache.org/jira/browse/SPARK-10848) where this topic was discussed. This logic has sense when you need to work with CSV file and there is no way to predict whether the field is nullable or not. 

## Workaround

Idea: get correct schema from one of avro files and then recreate dataframe based on correct schema.

{% highlight scala %}
val avroSchema = getSchema(avroFiles.head, spark)
val dfWithOptionalFields = readDF(spark, avroFiles)
val dfWithCorrectFields = dfWithOptionalFields.sqlContext.createDataFrame(dfWithOptionalFields.rdd, avroSchema)
{% endhighlight %}


