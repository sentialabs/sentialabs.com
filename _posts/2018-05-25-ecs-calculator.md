---
layout: post
title: Cost efficiency while using the containers
banner: /assets/posts/2018-05-25-ecs-calculator/ecs-banner.png
author:
  - veranikaisakova
  - hadoan
---

In the cloud, containers provide a containerised environment enabling your code to be built, shipped and run anywhere. This can be simply done by just running your code without setting up your operating system.

Coming with such benefit, AWS Elastic Container Services (ECS) is a a highly scalable, fast, container management service that makes easy to run your containerised code and applications across a managed cluster of EC2 instances. You will need to optimally scale and fit ECS containers in an EC2 instance for the efficient usage of your resources in the cloud. This means you should ensure that resources such as CPU and Memory units are enough for your largest container to scale out and efficiently reserved so that your containers can fill the instances in your deployment.

The table below provides the values of the CPU and Memory reservation for each EC2 instance type. You can use this information to pick optimal values for the CPU and Memory that will be assigned to your containers. This will be further explained in the examples that follow.

<div markdown="1" class="table-responsive ec2-table">

|Instance<br/>Type| CPU| RAM|
| :-------------: | :----: | :----: |
|t2.micro| 1024| 993|
|t2.small| 1024| 2001|
|t2.medium| 2048| 3952|
|t2.large| 2048| 7984|
|t2.xlarge| 4096| 16048|
|t2.2xlarge| 8192| 32176|
|----------------------|
|m4.large| 2048| 7984|
|m4.xlarge| 4096| 16048|
|m4.2xlarge| 8192| 32176|
|m4.4xlarge| 16384| 64417|
|m4.10xlarge| 40960| 161185|
|m4.16xlarge| 65536| 257953|
|----------------------|
|m3.medium| 1024| 3765|
|m3.large| 2048| 7480|
|m3.xlarge| 4096| 15040|
|m3.2xlarge| 8192| 30160|
|----------------------|
|c4.large| 2048| 3765|
|c4.xlarge| 4096| 7480|
|c4.2xlarge| 8192| 15040|
|c4.4xlarge| 16384| 30145|
|c4.8xlarge| 36864| 60385|
|----------------------|
|c3.large| 2048| 3765|
|c3.xlarge| 4096| 7480|
|c3.2xlarge| 8192| 15040|
|c3.4xlarge| 16384| 30145|
|c3.8xlarge| 32768| 60385|
|----------------------|
|r4.large| 2048| 15292|
|r4.xlarge| 4096| 30664|
|r4.2xlarge| 8192| 61408|
|r4.4xlarge| 16384| 122881|
|r4.8xlarge| 32768| 245857|
|r4.16xlarge| 65536| 491809|
|----------------------|
|r3.large| 2048| 15299|
|r3.xlarge| 4096| 30680|
|r3.2xlarge| 8192| 61442|
|r3.4xlarge| 16384| 122950|
|r3.8xlarge| 32768| 245996|
|----------------------|
|i2.xlarge| 4096| 30680|
|i2.2xlarge| 8192| 61442|
|i2.4xlarge| 16384| 122950|
|i2.8xlarge| 32768| 245996|
|----------------------|
|i3.large| 2048| 15292|
|i3.xlarge| 4096| 30664|
|i3.2xlarge| 8192| 61408|
|i3.4xlarge| 16384| 122881|
|i3.8xlarge| 32768| 245857|
|i3.16xlarge| 65536| 491809|
|----------------------|
|m5.large| 2048| 7690|
|m5.xlarge| 4096| 15586|
|m5.2xlarge| 8192| 31377|
|m5.4xlarge| 16384| 62960|
|m5.12xlarge| 49152| 189294|
|m5.24xlarge| 98304| 378665|
|----------------------|
|c5.large| 2048| 3714|
|c5.xlarge| 4096| 7634|
|c5.2xlarge| 8192| 15473|
|c5.4xlarge| 16384| 31152|
|c5.9xlarge| 36864| 70351|
|c5.18xlarge| 73728| 140780|
|----------------------|
|d2.xlarge| 4096| 30664|
|d2.2xlarge| 8192| 61408|
|d2.4xlarge| 16384| 122881|
|d2.8xlarge| 36864| 245857|

</div>

❗️You can also find this information in yaml format [here](/assets/files/2018-05-25-ecs-calculator/ec2-instance-list.yml) 👈

If the selected values for the CPU and Memory assigned to the container are not quotients of the CPU and Memory values that the instance provides, you will end up with empty blocks of unused resources. The solution is to make the optimal choice of the EC2 instance type with its budgeted CPU and Memory units to efficiently fit ECS containers in the instance based on the data from the table. Therefore, it is important to first have some insight into the limits of CPU and Memory reservation.

The CPU assignment is a soft limit which means that containers can burst above their provision. One container can burst above its allocated units if no other containers are taking up resources. Besides, containers also share their unallocated CPU units with other containers on the instance with the same ratio as their allocated amount.

It is also useful to understand how your container can effectively use the memory resources.The ECS memory resource model is more complex then the CPU resource model and allows you to configure both memory reservations and memory limits. You can see the whole concept below:

![memory](/assets/posts/2018-05-25-ecs-calculator/memory_resources.png)

Let's assume your container normally uses 128 MBs of memory, but occasionally bursts to 256 MBs of memory for short periods of time. As an option, you can set a Memory Reservation of 128 MBs, and a Memory Limit of 300 MBs. This configuration would allow the container to only reserve 128 MBs of memory from the resources on the container instance, but also allows the container to consume more memory resources when needed. Another option is to only use the hard limit for the memory assignment. This way you can bound each container to a specific value and avoid containers fighting for resources and possibly crashing because of lack of memory units. Hard memory bound requires extensive profiling of your containers for optimal resource allocation.

## How the data can help you to optimally scale and fit ECS containers in an EC2 instance?

Based on the soft/hard limit of CPU/Memory reservation, the approach of solving our current issue is taken according to the following steps.
* Obtaining the number of the Memory units required for all your containers.
* Optimally choosing the EC2 instance type that can host your largest container based on the memory units it requires.
* Calculating CPU units needed for your all your containers.

The related examples are given below.

## Examples

Imagine that your largest container requires 332 memory units. If you choose `t2.micro` as the EC2 instance type which offers 993 memory units, you can fit 2 containers in the instance (993/332 = 2.99 containers). However, if you choose `t2.small` which offers 2001 memory units, the memory resource would be more efficiently used because 2001/332 = 6.02 containers and not using the leftover 0.02 containers would be less wasteful than doing this with the leftover 0.99 containers. In this way, your containers fit more efficiently in the `t2.small` instance than in the `t2.micro` instance. The next question is how can we calculate CPU for your container. The amount for CPU units you can easily calculate using this ratio:

![ratio](/assets/posts/2018-05-25-ecs-calculator/ratio_cpu_ram.png)

> So, (332 Mbs * 1024 CPU units) / 2001 Mbs = 169 CPU units.

In another scenario, your largest container is provided with 900 memory units and you would like to pick an instance that can host 4 containers. You can fit this amount of containers in the `t2.medium` instance which reserves 3952 Memory units (3952/900 = 4.39 containers). Also, the `t2.large` instance which reserves 7984 Memory units makes it possible for 4 containers to fit in as well (7984/900 = 8.87 containers). Then your containers will fit more efficiently in the `t2.medium` instance than in the `t2.large` instance because we will have much less unused resources. Let's calculate the CPU units then.

> (900 Mbs * 2048 CPU units) / 3952 Mbs =  466 CPU units.

The examples above are indeed based on the important idea: available resources should be used up for the sake of the efficiency. Therefore, in case you have troublesome after some time to find an EC2 instance type in the table that suits your needs, there are two options being advised:
* Providing more Memory units if available to your largest container so that your unused resources will be efficiently decreased.
* Using the `m4.large` instance for it overall reserves good amount of CPU/Memory units (2048/7984) for your containers and is used for general purposes.

## Summary

We hope that this article would be useful in your day-to-day practice of optimally scaling and fitting ECS containers in the EC2 instances of your choice and help you and your company to save money 💰
