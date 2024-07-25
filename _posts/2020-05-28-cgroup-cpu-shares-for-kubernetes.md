---
layout: post
title: 'CPU Shares in Kubernetes'
author: Christopher Batey
date: '2020-05-28'
comments: true
tags:
- docker 
- kubernetes
- containers
- cpu_shares
- cgroups
---

In this article we'll explain how **cpu_shares** are used when setting Kubernetes requests and limits.
You should first understand **cpu_shares** which are explained in [CPU Shares](/cgroup-cpu-shares-for-docker.html).

Kubernetes has its own abstraction for CPUs called **cpus**. A Kubernetes resource can be set as a 
**request** or a **limit**. When set as a request then **cpu_shares** are used to specify how much of the available
CPU cycles a container gets. A Kubernetes **limit** results in **cpu_quota** and **cpu_period** being used. 

### CPU Shares value

Kubernetes translates 1000 millicores or 1 core as 1024 **cpu_shares**. For example 1500 millicores are translated into
1536 **cpu_shares**. We learned in the previous post on [CPU Shares](/cgroup-cpu-shares-for-docker.html) that a **cpu_share**
value doesn't mean anything in isolation. Shares are a relative value to the other containers running on the same machine. That's
where the Kubernetes scheduler comes in.

### Scheduling

The Kubernetes scheduler uses the request value for scheduling. The containers scheduled to a machine cannot have a **cpu** 
count greater than the actual number of cpus on a machine.
This means that **cpu_shares** when used in Kubernetes gives more information than when they are used in isolation.
The total of the **cpu_shares** for all the containers on a Kubernetes node cannot exceed **\<nr of CPUs\> x 1024**.
This results in the **request** value acting as a minimum amount of CPU time for a container. When there is no contention 
for CPU a container can use more CPU than its **request**. The scenario described in the 
[previous post](/cgroup-cpu-shares-for-docker.html) cannot happen in a Kubernetes cluster. 

<img src="/assets/cpu_shares/one-cpu.png" class="img-fluid mt-1 pl-5 pr-5" />

In this scenario the machine has 1 CPU. The scheduler prevents it as the number of shares is greater than 1024. 

### No request

Containers without a CPU request get 0 **cpu_shares**. They can be assigned to any machine, even a node hosting containers with
**\<nr of CPUs\> x 1024** shares. This does not violate the scheduling described above as the container has 0 **cpu_shares**.
Under high CPU contention the containers without **requests** will get no CPU time.

### Take aways

The Kubernetes scheduler along with **requests** allows reasoning about the minimum amount of CPU time a container will get.
The same is not true when using **cpu_shares** without a scheduler limiting the number of shares assigned to a machine.

All but the least important containers should have a **request** set otherwise the container can be starved of CPU time indefinitely.
Batch processing without an SLA is a valid use case for not using a CPU **request**. That way the batch process will use up any free
CPU resources and be throttled when real-time work executes.

{% include jd-course.html %}

