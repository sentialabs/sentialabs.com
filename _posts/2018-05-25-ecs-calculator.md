---
layout: post
title: Cost efficiency while using the containers
banner: /assets/posts/2018-05-25-ecs-calculator/ecs-banner.png
author:
  - veranikaisakova
  - hadoan
---

In the cloud, containers provide a containerised environment enabling your code to be built, shipped and run anywhere. This can be simply done by just running your code without setting up your operating system.

Coming with such benefit, AWS Elastic Container Services (ECS) is a a highly scalable, fast, container management service that makes easy to run your containerised code and applications across a managed cluster of EC2 instances. You will need to optimally scale and fit ECS containers in an EC2 instance for the efficient usage of your resources in cloud. This means that the CPU and memory capacity of the EC2 instance should be used up to be distributed over the possible maximum number of containers that can fit in the EC2 instance. You can have a quick access to data of the CPU and memory reservation for each EC2 instance type using the table with all information below.

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

How the data can help you to optimally scale and fit ECS containers in an EC2 instance?
==========================================================================================

The number of CPU units specifies the minimum amount of CPU reserved for a container. The CPU units are effectively allocated in containers on an EC2 instance. One container can burst above its allocated units if no other containers are taking up resources. **For instance**, if you have two tasks running on a `t2.medium`, each with 1024 CPU units budgeted for its respective container, one task could take up the whole 2048 CPU units given that the other task is inactive.

Besides, containers also share their unallocated CPU units with other containers on the instance with the same ratio as their allocated amount. **For example**, if you have two tasks running on a `t2.medium` each with 0 CPU units, each container of each task will effectively get 1024 CPU units since no other container has reserved them.

Whereas the CPU units can be reserved with soft limit because containers can burst above their provision, the Memory reservation has to stay within the allocated amount. If your container attempts to exceed the memory, the container is killed 😿 (this may require some testing to get it right).

It's also useful to understand how your container can effectively use the memory resources.The ECS memory resource model is more complex then the CPU resource model and allows you to configure both memory reservations and memory limits. The whole concept you can see below:

![memory](/assets/posts/2018-05-25-ecs-calculator/memory_resources.png)

Let's assume your container normally uses 128 MBs of memory, but occasionally bursts to 256 MBs of memory for short periods of time. As an option, you can set a Memory Reservation of 128 MBs, and a Memory Limit of 300 MBs. This configuration would allow the container to only reserve 128 MBs of memory from the remaining resources on the container instance, but also allows the container to consume more memory resources when needed. The 3rd scenario where you just configure only Hard Memory Limit with no Memory Reservation. In this case the ECS container instance will deduct configured memory limit from the container instance memory resources. We think that it's better to use exactly this option because by specifying only a Hard Memory Limit for your tasks you avoid running out of memory (ECS stops placing tasks on the instance, and docker kills any containers that try to go over the hard limit), i.e. Hard Memory Limit exists for the optimal fit containers in the instance.

Examples
====================

Imagine that you need a minimum of 332 MBs of memory (memory units) for your container. If you choose `t2.micro` as the EC2 instance type which offers 993 memory units, you can fit 2 containers in the instance (993/332 = 2.99 containers). However, if you choose `t2.small` which offers 2001 memory units, the memory resource would be more efficiently used because 2001/332 = 6.02 containers and not using the leftover 0.02 containers would be less wasteful than doing this with the leftover 0.99 containers. The next question is how can we calculate CPU for your container. The amount for CPU units you can easily calculate using this ratio:

![ratio](/assets/posts/2018-05-25-ecs-calculator/ratio_cpu_ram.png)

> So, (332 Mbs * 1024 CPU units) / 2001 Mbs = 170 CPU units.

Summary
====================

We hope that this article would be useful in your day-to-day practice and help you to save money 💰
