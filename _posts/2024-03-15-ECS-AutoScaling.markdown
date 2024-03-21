---
layout: post
title:  "AWS ECS AutoScaling and CPU Metrics"
date:   2024-03-07 12:13:26 +0000
tags: [AWS,ECS]
---


## AutoScaloing 

We will cover : 
CPU 
Memory

## Service CPU Scaling:
Scaling based on service CPU utilizes CloudWatch alarms associated with the ECS service to trigger scaling actions. When the average CPU utilization of all tasks in the service exceeds a specified threshold, ECS can automatically scale the service.

**Advantages:**
Simple to set up and manage, especially for applications with multiple tasks running in a service.Automatically considers the aggregate CPU usage of all tasks in the service.

**Disadvantage**
Is it very important is to evenly distribute the workload across our tasks to avoid hot tasks and cold tasks. What can end up happening is you have one client or one connection that is doing a lot of volume over the connection versus other connections are much lower traffic. And what that ends up with is you end up with hot tasks that are hitting the resource limits while other tasks are under utilized, they’re cold.

The danger there is you can end up with aggregate metrics that still look good. It looks like the metrics are within bounds and it doesn’t look like there’s any need to scale. Or a Scale in activity can happen killing an active hot task before the task is finished.


### Task CPU Scaling:
Scaling based on task CPU involves using CloudWatch alarms associated with individual ECS tasks to trigger scaling actions. 

Each task has its own CPU alarm, and scaling decisions are made based on the CPU utilization of individual tasks.

#### Scaling with Task CPU:

Scaling Up:

Trigger: An individual task's CPU utilization breaches its upper threshold.
Action: Add a new task to handle the increased workload.
Scaling Down:

Trigger: A task's CPU utilization falls below its lower threshold and there are enough healthy tasks to handle the remaining workload.
Action: Terminate the underutilized task.

**Advantages**:
Provides more fine-grained control over scaling decisions, as it considers the CPU usage of each task independently.

**Disadvantage:** 
Expensive to use Container metrics 

## Testing
Using Amazon ECS Exec to access your containers on AWS Fargate and Amazon EC2

aws ecs execute-command  \
    --region $AWS_REGION \
    --cluster ecs-exec-demo-cluster \
    --task ef6260ed8aab49cf926667ab0c52c313 \
    --container nginx \
    --command "/bin/bash" \
    --interactive



Use Stress tools :

    apt-get update && apt-get install -y stress

CPU:
stress --cpu 8 --timeout 1200s & top

Memory 
stress --vm 2 --vm-bytes 250M --timeout 300s

Both :
stress --cpu 8 --timeout 600s & stress --vm 1 --vm-bytes 512M --timeout 600s top
 
stress --cpu 1 --vm 1 --vm-bytes 1G --timeout 600s

**Findings** 

Task hits 100%
CloudWatch Metrics : 
Service Average CPU utilisation : 100%
Service Mamimum CPU utilisation : 100%

![Metrics](/assets/img/autoscalling/twotasks.jpg)

Scalling is triggered based on Average 

Second task doing nothing ( the aggratated CPU Drops)
![Metrics](/assets/img/autoscalling/onetask.jpg)

CloudWatch Metrics : (the average ia acumlitate accross all ataks so we have ahost and cold task )
Service Average CPU utilisation : 50%
Service Mamimum CPU utilisation : 100%

**Scale in now happens as the averave CPU is now lowered** 

The oldest task is killed first ( this is the actually a Hot task that is running the process)


### Solutions

Auto Scalling scaling in protection 
https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_UpdateTaskProtection.html

Updating 

Reference : https://aws.amazon.com/blogs/containers/new-using-amazon-ecs-exec-access-your-containers-fargate-ec2/

