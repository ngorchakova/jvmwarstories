---
layout: post
title: Cloud watch events for glue jobs
date: 2020-06-16 00:00:00 +00:00
tags: [spark, aws, glue, monitoring, cloudwatch]
---
## Problem
We have several Spark jobs running as Glue jobs in AWS. The incident system is Pager Duty. The goal was to have an alerting in case of failure of the glue job. So, we wanted to deliver a cloud watch event of the glue job failure to Pager Duty.
    

## Solution
Unfortunately, there is no out of the box integration with Pager Duty for AWS cloud watch events. The solution for us was to create a lambda job that consumes SNS topic with some json information about incidents and pushes them to Pager Duty. The lambda job itself is quite simple. Here I will focus on how we deliver information about Glue job failure to SNS topic.

Each time glue job fails new cloud watch event is triggered with the corresponding status

Example of such message:
{% highlight json %} 
{
  "version": "0",
  "id": "message_long_id",
  "detail-type": "Glue Job State Change",
  "source": "aws.glue",
  "account": "xxxxxxxxxxxx",
  "time": "2020-01-01T00:00:00Z",
  "region": "eu-west-1",
  "resources": [],
  "detail": {
    "jobName": "job_name",
    "severity": "ERROR",
    "state": "FAILED",
    "jobRunId": "jr_long_id",
    "message": " reason of failure"
  }
}

{% endhighlight %}

So, the goal was to push a proper message to SNS. That was achieved by CloudWatch event rule. As the job that consumes messages from SNS was designed to be generic, we expected that the message is in json format. Json messages should contain a field "subject" that would be used as a title of an incident.


## Terraform script to provision cloud watch event rule


{% highlight json %}
resource "aws_cloudwatch_event_rule" "glue_monitoring_rule" {
  name = "glue-job-failure-alerting"
  description = "Alerts on each glue job failure"

  event_pattern = <<PATTERN
{
  "source": [
    "aws.glue"
  ],
  "detail-type": [
    "Glue Job State Change"
  ],
  "detail": {
	"state": ["FAILED"]
  }
}
PATTERN
}

data "aws_sns_topic" "alert_sns" {
  name = "${var.alert-sns-name}"
}

resource "aws_cloudwatch_event_target" "rule_target_sns" {
  rule = "${aws_cloudwatch_event_rule.console.name}"
  arn = "${data.aws_sns_topic.aleret_sns.arn}"
  input_transformer = {
    input_paths = {
      "jobName" = "$.detail.jobName",
      "event" = "$"
    }
    input_template = <<TEMPLATE
{
	"subject": <jobName>,
	"event": <event>
}
TEMPLATE
  }
}
{% endhighlight %}