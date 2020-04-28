---
layout: post
title: Define user in hue on EMR provisioning
date: 2020-04-29 00:00:00 +00:00
tags: [aws glue, hue]
---
## Environment
EMR cluster (I tested on 5.20.0 and 5.29.0) 

## Problem
Every time the new EMR cluster is provisioned during the first login to HUE I was forced to provide user name and password. To be sure that the password is the same, so the whole team can use it, I automated the user setup during EMR provisioning.  


##Solution

create a script file setup-default-user.sh and upload it to s3. The script below create a user `hadoop` with password `Pa$$word123`.

{% highlight bash %}
    #!/usr/bin/expect
    spawn sudo /usr/lib/hue/build/env/bin/hue createsuperuser
    expect "Username (leave blank to use 'root'):"
    send "hadoop\r"
    expect "Email address:"
    send "\r"
    expect "Password:"
    send "Pa\$\$word123\r"
    expect "Password (again):"
    send "Pa\$\$word123\r"
{% endhighlight %}

After EMR cluster is provisioned add a step. As we provision an EMR cluster via terraform, the script looks like:
{% highlight bash %}
     terraform init ...
     terraform apply
     clusterId="$(terraform output clusterId)"
     scriptRunner="s3://${region}.elasticmapreduce/libs/script-runner/script-runner.jar"
     aws emr add-steps --cluster-id "${clusterId}" --region "${region}" --steps Type=CUSTOM_JAR,Name=SetupHueDefaultUser,ActionOnFailure=CONTINUE,Jar="${scriptRunner}",Args="${s3scripts}/hue/setup-default-user.sh","${env}"
{% endhighlight %}

ClusterId is defined as an output of the terraform script
{% highlight json %}
    output "clusterId" {
      value = "${aws_emr_cluster.emr.id}"
    }
{% endhighlight %}
