---
layout: post
title: ECS container instance scaling the proper way
banner: /assets/posts/2018-05-16-Custom-ECS-Container-Instance-Scaling-Metric/ecs.png
author:
  - kbessas
---

When managing your own cluster in ECS, there are 2 metrics you can use to scale your instances. Namely, these are the CPU and Memory reservation. For reference, I would like to mention here the simple mechanics behind them. Each of these metrics represent the percentage of CPU and respectively Memory units that are reserved by running tasks in the cluster.

Scaling your instances in ECS optimally is not an easy or straightforward task. At least not when you are trying to do so using only the previous metrics that are available to you through CloudWatch. The problem originates from the fact that you have 2 metrics to consider.

**It is impossible to scale using 2 metics if they are uncorrelated to each other.**

So how can you solve this problem?

Correlate all the data
----------------------

At Sentialabs we have a great [post](http://www.sentialabs.io/2018/05/25/ecs-calculator.html) about how you can use CPU and Memory values for your services/containers proportional to the available values depending on the instance class you are using for your ECS cluster. When you do so, you automatically create a correlation for the 2 metrics. Subsequently, you can just use one of the 2 metrics to scale your container instances. The previous depends on depends on the workload your containers are running, it is a bit restrictive and requires effort and recalculation for the values when you want to change the class of your instances in the cluster.

But what if you can't follow the previous guidelines especially because your container profiling has shown that they require very extravagant custom values?

Create a custom metric
----------------------

There are various approaches described in a variety of blog posts on how you can implement a custom scaling metric for container instance scaling on ECS. At Sentia, we reviewed and benchmarked many of these solutions for their dynamics and efficiency. We ended up creating a custom scaling metric of our own.

What is the difference?

A typical scaling metric outputs values in CloudWatch, and then you can use thresholds in the alarms to trigger actions. The problem is that ECS is a very dynamic service which means that the thresholds may need to be recalculated when services come and go. Our solution is to let the calculations be performed by the script that does all the correlation for the data. And the medium for this is of course Lambda.


Put everything together in Lambda
---------------------------------

```python
__copyright__ = """
    Copyright 2018 Sentia <www.sentia.nl>
    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at
       http://www.apache.org/licenses/LICENSE-2.0
    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
"""
__license__ = "Apache 2.0"


import boto3
import os
import datetime
import dateutil
import logging


def lambda_handler(event, context):
    ecs_cluster_name = event['ecs_cluster_name']
    region_name = os.environ['AWS_REGION']
    scalability_index = event['scalability_index']

    # ec2client = boto3.client('ecs', region_name=os.environ['AWS_REGION'])
    ecs = boto3.client('ecs', region_name=region_name)
    cw = boto3.client('cloudwatch', region_name=region_name)
    logger = logging.getLogger()
    logger.setLevel(logging.INFO)

    ##############################
    # Calculate services metrics
    services_arns = ecs.list_services(cluster=ecs_cluster_name)['serviceArns']
    services = ecs.describe_services(
        cluster=ecs_cluster_name, services=services_arns)['services']

    # Largest container requirements
    min_required_cpu_units = 0
    min_required_mem_units = 0
    # Total reservation across instances
    total_registered_cpu_units = 0
    total_registered_mem_units = 0
    # Total nominal units per instance
    cpu_units_per_instance = 0
    mem_units_per_instance = 0
    # Free space across available instances, per instance
    remaining_cpu_units_per_instance = []
    remaining_mem_units_per_instance = []
    # Schedulable largest container
    schedulable_largest_containers_by_cpu = 0
    schedulable_largest_containers_by_mem = 0

    for service in services:
        # TODO: enrich finctionality for quick scaling based on desired count,
        # or running count
        running_count = service['runningCount']
        desired_count = service['desiredCount']
        container_definitions = ecs.describe_task_definition(
            taskDefinition=service['taskDefinition']
        )['taskDefinition']['containerDefinitions']
        for container_definition in container_definitions:
            min_required_cpu_units = max(
                min_required_cpu_units, container_definition['cpu'])
            min_required_mem_units = max(
                min_required_mem_units, container_definition['memory'])

    ##############################
    # Calculate cluster metrics
    container_instance_arns = ecs.list_container_instances(
        cluster=ecs_cluster_name)['containerInstanceArns']
    container_instances = ecs.describe_container_instances(
        cluster=ecs_cluster_name,
        containerInstances=container_instance_arns)['containerInstances']

    for container_instance in container_instances:
        remaining_resources = {
            resource['name']: resource
            for resource in container_instance['remainingResources']}
        schedulable_largest_containers_by_cpu += int(
            remaining_resources['CPU']['integerValue']
            / min_required_cpu_units)
        schedulable_largest_containers_by_mem += int(
            remaining_resources['MEMORY']['integerValue']
            / min_required_mem_units)
        for resources in container_instance['registeredResources']:
            if 'CPU' in resources.values():
                total_registered_cpu_units += resources['integerValue']
            if 'MEMORY' in resources.values():
                total_registered_mem_units += resources['integerValue']
        for resources in container_instance['remainingResources']:
            if 'CPU' in resources.values():
                remaining_cpu_units_per_instance.append(
                    resources['integerValue'])
            if 'MEMORY' in resources.values():
                remaining_mem_units_per_instance.append(
                    resources['integerValue'])

    cpu_units_per_instance = total_registered_cpu_units // len(
        container_instances)
    mem_units_per_instance = total_registered_mem_units // len(
        container_instances)

    cpu_scale_in_threshold = int(
        cpu_units_per_instance / min_required_cpu_units) + scalability_index
    mem_scale_in_threshold = int(
        mem_units_per_instance / min_required_mem_units) + scalability_index

    # Check for required scaling activity
    # {"-1": "scale in", "0": "no scaling activity", "1": "scale out"}
    if min(schedulable_largest_containers_by_cpu,
           schedulable_largest_containers_by_mem) < scalability_index:
        logger.info(('A total of (CPU: %d, MEM: %d) can be scheduled based on '
                     'each metric. This is less than the scalability index '
                     '(%d)') % (
            schedulable_largest_containers_by_cpu,
            schedulable_largest_containers_by_mem,
            scalability_index))
        logger.info(('Scale out is required, but I can only update the  '
                     'metric from my point of view.'))
        requires_scaling = 1
    elif (schedulable_largest_containers_by_cpu >= cpu_scale_in_threshold and
          schedulable_largest_containers_by_mem >= mem_scale_in_threshold):
        logger.info(('A total of (CPU: %d, MEM: %d) of the largest containers '
                     ' can be scheduled based on each metric. This is larger '
                     'or equal to the threshold of (CPU: %d, MEM: %d).') % (
            schedulable_largest_containers_by_cpu,
            schedulable_largest_containers_by_mem,
            cpu_scale_in_threshold,
            mem_scale_in_threshold))
        logger.info(('Scale in is required, but I can only update the metric '
                     'from my point of view.'))
        requires_scaling = -1
    else:
        logger.info('Everything looks great. No scaling actions required.')
        requires_scaling = 0

    cw.put_metric_data(Namespace='AWS/ECS',
                       MetricData=[{
                           'MetricName': 'RequiresScaling',
                           'Dimensions': [{
                               'Name': 'ClusterName',
                               'Value': ecs_cluster_name
                           }],
                           'Timestamp': datetime.datetime.now(
                                            dateutil.tz.tzlocal()),
                           'Value': requires_scaling
                       }])

    return {}
```

This script takes as input from the event 2 values:
  * An ECS cluster name
  * A scalability index

The ECS cluster name is used to specify the cluster against which the metrics will be calculated. The scalability index is used to specify how fast the cluster can scale out with new containers. Scalability index guarantees chunks of free space for "scalability index" amount of the largest containers.

How does the script work?

The script calculates several values for the designated cluster in regard to the ECS units. Based on these values, it sets a cloudwatch metric in the namespace AWS/ECS with name RequiresScaling.
  *  0  -  No scaling required
  *  1  -  Scale out required
  * -1  -  Scale in required

A scale out would happen when:
  * There are not sufficient chunks of space based on the scalability
    index.

In order for the cluster to scale in, the previous scale out rules should not be violated after a complete instance is removed from the cluster. This is ensured by checking that there are sufficient chunks of free space even after the reorganization in ECS after a scale in activity.

How does it look like in practice?
----------------------------------

With the above solution you no longer need to be careful about the cpu and memory units you assign to your services. You can profile your container and deploy them in the stack without having to worry about the stability of the scaling algorithm or the thresholds that you have set in the code.

On top of this you get a scalability index which directly translates to a prioritization of cost or performance priority scaling activity. With a smaller scalability index you have less space for containers to scale out faster, but also less unused space within your instances. When you increase the scalability index, you have more space for containers to scale out faster, deployments that need to replace a lot of containers to finish faster but of course you are less cost effective.

In the following captions you can find some screenshots of how the algorithm operates in practice. Although the actual data for triggering the scaling activities is missing, it serves as an example of how the above code snippet will work for your account.

![Scaling Metric Example](/assets/posts/2018-05-16-Custom-ECS-Container-Instance-Scaling-Metric/requires_scaling_metric.png)
![Scaling Activities 1](/assets/posts/2018-05-16-Custom-ECS-Container-Instance-Scaling-Metric/scaling_1.png)
![Scaling Activities 2](/assets/posts/2018-05-16-Custom-ECS-Container-Instance-Scaling-Metric/scaling_2.png)

As a reference, please find below the CloudFormation code that shows an example of how the CloudWatch alarms can be set up around the RequiresScaling metric.

```json
"EcsClusterScaleInAlarm": {
  "Properties": {
    "ActionsEnabled": true,
    "AlarmActions": [{
      "Ref": "EcsClusterScaleInPolicy"
    }],
    "ComparisonOperator": "LessThanOrEqualToThreshold",
    "Dimensions": [{
      "Name": "ClusterName",
      "Value": {
        "Ref": "EcsClusterCluster"
      }
    }],
    "EvaluationPeriods": 5,
    "MetricName": "RequiresScaling",
    "Namespace": "AWS/ECS",
    "Period": 60,
    "Statistic": "Maximum",
    "Threshold": -1,
    "TreatMissingData": "notBreaching",
    "Unit": "None"
  },
  "Type": "AWS::CloudWatch::Alarm"
},
"EcsClusterScaleInPolicy": {
  "Properties": {
    "AdjustmentType": "ChangeInCapacity",
    "AutoScalingGroupName": {
      "Ref": "EcsClusterInstanceAutoScalingGroup"
    },
    "Cooldown": 300,
    "ScalingAdjustment": -1
  },
  "Type": "AWS::AutoScaling::ScalingPolicy"
},
"EcsClusterScaleOutAlarm": {
  "Properties": {
    "ActionsEnabled": true,
    "AlarmActions": [{
      "Ref": "EcsClusterScaleOutPolicy"
    }],
    "ComparisonOperator": "GreaterThanOrEqualToThreshold",
    "Dimensions": [{
      "Name": "ClusterName",
      "Value": {
        "Ref": "EcsClusterCluster"
      }
    }],
    "EvaluationPeriods": 1,
    "MetricName": "RequiresScaling",
    "Namespace": "AWS/ECS",
    "Period": 60,
    "Statistic": "Maximum",
    "Threshold": 1,
    "TreatMissingData": "notBreaching",
    "Unit": "None"
  },
  "Type": "AWS::CloudWatch::Alarm"
},
"EcsClusterScaleOutPolicy": {
  "Properties": {
    "AdjustmentType": "ChangeInCapacity",
    "AutoScalingGroupName": {
      "Ref": "EcsClusterInstanceAutoScalingGroup"
    },
    "Cooldown": 120,
    "ScalingAdjustment": 1
  },
  "Type": "AWS::AutoScaling::ScalingPolicy"
}
```

Conclusion
----------

Container instances scaling for ECS can be a challenging topic. By harvesting the power of Lambda and outsourcing the decision for optimal scheduling to a script, you end up with a process resilient to human error. You are also protecting your infrastructure against changes in the number of services and containers within your cluster.
