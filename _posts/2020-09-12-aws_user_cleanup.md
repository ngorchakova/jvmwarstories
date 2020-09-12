---
layout: post
title: Python script to delete user with all dependencies
date: 2020-09-12 00:00:00 +00:00
tags: [aws,python, boto3, iam]
---
## Problem
After the initial adoption of AWS there were a lot of users created manually, with policies, groups and access keys. So, the goal was to have possibility to automatically remove them.  
    

## Solution
Small python script with boto3 was done. 

The input is:
* user  - String representation of user name
* iam - iam client from boto3 `boto3.client('iam')`


{% highlight python %} 

def deleteUser(user, iam):
    print("detele " + user)

    # delete all access keys
    access_keys = iam.list_access_keys(UserName=user)
    for key_entry in access_keys.get("AccessKeyMetadata"):
        access_key = key_entry.get("AccessKeyId")
        print(access_key)
        # delete this key
        iam.delete_access_key(AccessKeyId=access_key,UserName=user)

    # remove all groups
    userGroups = iam.list_groups_for_user(UserName=user)
    for groupName in userGroups['Groups']:
        print(groupName['GroupName'])
        # delete group
        iam.remove_user_from_group(GroupName=groupName['GroupName'], UserName=user)

    #remove policies
    for policy in iam.list_user_policies(UserName=user).get("PolicyNames"):
        print policy
        iam.detach_user_policy(UserName = user, PolicyArn=policy)

    print iam.list_attached_user_policies(UserName=user)
    for policy in iam.list_attached_user_policies(UserName=user).get("AttachedPolicies"):
        print policy
        iam.detach_user_policy(UserName = user, PolicyArn=policy.get("PolicyArn"))
    # remove user
    iam.delete_user(UserName=user)

{% endhighlight %}

