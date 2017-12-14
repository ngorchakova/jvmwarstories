---
layout: post
title: Confluent vs Hortonworks avro compatibility check
date: 2017-12-14 00:00:00 +00:00
tags: [avro, confluent, hortonworks]
---
## Versions
Confluent schemaRegistry - 3.1.2
Hortonworks schemaRegistry - 0.3.0

## Case
By mistake new required field was added to avro schema and confluent schema registry didn't complained about that. Later, during migration schemas to Hotronworks schemaRegistry, this problem arose.

Simplified schema examples:

{% highlight json %}
{
  "type": "record",
  "name": "Event",
  "namespace": "test.namespace",
  "fields": [
    {
      "name": "topOptionalField",
      "type": [
        "null",
        {
          "type": "record",
          "name": "optionalField",
          "fields": [
            {
              "name": "required1",
              "type": "int"
            }
          ]
        }
      ],
      "default": null
    }
  ]
}
{% endhighlight %}

next version of the schema with one more required field `required2`

{% highlight json %}
{
  "type": "record",
  "name": "Event",
  "namespace": "test.namespace",
  "fields": [
    {
      "name": "topOptionalField",
      "type": [
        "null",
        {
          "type": "record",
          "name": "optionalField",
          "fields": [
            {
              "name": "required1",
              "type": "int"
            },
            {
              "name": "required2",
              "type": "int"
            }
          ]
        }
      ],
      "default": null
    }
  ]
}
{% endhighlight %}

check compatibility

{% highlight scala %}

public void testRealSchemas() throws IOException {
      final String fromSchemaString = IOUtils.toString(TestAvro.class.getResourceAsStream("/avro/simple_from.json"), "UTF-8");
      final String toSchemaString = IOUtils.toString(TestAvro.class.getResourceAsStream("/avro/simple_to.json"), "UTF-8");
      final Schema fromSchema = new Schema.Parser().parse(fromSchemaString);
      final Schema toSchema = new Schema.Parser().parse(toSchemaString);

      final AvroCompatibilityChecker confluentChecker = AvroCompatibilityChecker.BACKWARD_CHECKER;

      System.out.println(confluentChecker.isCompatible(fromSchema, toSchema)); //true
      System.out.println(confluentChecker.isCompatible(toSchema, fromSchema)); //true

      System.out.println(new AvroSchemaProvider().checkCompatibility(fromSchemaString,
              toSchema.toString(),
              SchemaCompatibility.BACKWARD)); //true
      System.out.println(new AvroSchemaProvider().checkCompatibility(toSchemaString,
              fromSchema.toString(),
              SchemaCompatibility.BACKWARD)); //false
}
{% endhighlight %}

## Facts
There is a [ticket](https://github.com/confluentinc/schema-registry/issues/391) in confluent schema registry. According to it, they use avro library 1.8.1. And there is a bug in [avro jira](https://issues.apache.org/jira/browse/AVRO-1883). The bg in avro was fixed in 1.8.2. And this version of libarary is used for newest version of confluent schema registry.
Hortonworks implements its own validator for avro schemas [AvroSchemaValidator](https://github.com/hortonworks/registry/blob/master/schema-registry/common/src/main/java/com/hortonworks/registries/schemaregistry/avro/AvroSchemaValidator.java). 




