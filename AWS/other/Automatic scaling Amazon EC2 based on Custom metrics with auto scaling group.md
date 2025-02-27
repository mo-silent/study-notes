# Automatic scaling Amazon EC2 based on Custom metrics with auto scaling group

## What is auto scaling group?

An **Auto Scaling group** (ASG) is a feature provided by Amazon Web Services (AWS) that allows people to automatically manage and scale a collection of **Amazon EC2 instances** (virtual servers) based on predefined conditions.

## Why we use Custom metrics in auto scaling group?

Auto scaling group only provide four system metrics policies that is Average CPU utilization, Average network in(bytes), Average network out(bytes) and Application Load Balancer request count per target. 

But in our business scenario, these four system policies are not suitable. And then, the metrics provided by Amazon cloudwatch also don't meet our needs.

So we need to use custom metrics to manager the scaling policy in ASG.

