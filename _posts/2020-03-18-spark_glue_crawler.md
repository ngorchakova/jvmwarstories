---
layout: post
title: Glue crawler issues during spark job run
date: 2020-03-18 00:00:00 +00:00
tags: [aws glue, spark]
---
## Environment
We have our data lake running in AWS. There is some data on s3 in parquet files. This data is processed later by spark job and pushed back to s3.

The goal was to run a [glue crawler](https://docs.aws.amazon.com/glue/latest/dg/add-crawler.html) on top of the processed parquet files to build the data catalog. Later the tables can be used or in [Athena](https://docs.aws.amazon.com/athena/latest/ug/what-is.html) or in [Redshift Spectrum](https://docs.aws.amazon.com/redshift/latest/dg/c-getting-started-using-spectrum.html). The data has a quite popular structure on s3: `/processed/data_source/year=2020/month=03/day=18/files.parquet`. So we expected to have one table per data_source.  

## Problem
The crawler was set up to run daily and the spark job that process data also. While they didn't intersect everything was perfect. 

After some time we ran reprocessing and a spark job was working for several days. So a glue crawler and a spark job were working at the same moment. After that, we found out a huge mess in the glue tables. 

There were:
* table per day (i.e. day_01)
* table per month (i.e. month_01)
* event table per file (i.e. part_00000_snappy_parquet )

I compared all the tables definition and it appeared to be the same everywhere. The next step was to look in the glue crawler logs. There were ERRORS like:

`[7803bfa9-8f3b-42c3-b1e4-f57b8b42f656] ERROR : Error The specified key does not exist. (Service: Amazon S3; Status Code: 404; Error Code: NoSuchKey; Request ID: 2C085FD55CD5A0EF; S3 Extended Request ID: 61tJhfQzY3XnKejF3UeVd/X8gG82L115MGspPeZ1OtMWFBW2GlQOmn9te/5X8lCyhdbOa0jHHm4=) retrieving file at s3://xxx/processed/data_source/year=2020/month=01/day=03/_temporary_$folder$. Tables created did not infer schemas from this file.`

That was a huge hint: just add ignore of temporary files to glue crawler. That was immediately done.

##Solution

{% highlight json %}
resource "aws_glue_crawler" "processed-glue-crawler" {
  database_name = "${aws_glue_catalog_database.processed.name}"
  name = "processed-data--crawler"
  role = "${var.glue_iam_role_arn}"
    schedule = "cron(0 1 * * ? *)"
    
  s3_target {
      path = "${local.processed_data_path}"
      exclusions = ["**_temporary**"]
  }
}
{% endhighlight %}

Just from my point of view this is a bug in Glue Crawler. According to the documentation it creates a new table when the columns differ, but not when it cannot read some file.