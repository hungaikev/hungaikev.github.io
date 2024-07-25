---
layout: post
title: 'CPU Quota for Docker and Kubernetes'
author: Christopher Batey
comments: true
date: '2020-05-28'
tags:
- docker 
- kubernetes
- containers
- cpu_quota
- cgroups
---

In this article we'll explain CPU quota and period, so you can understand how to set them in Docker.

**CPU quota (cpu_qota)** is a feature of **Linux Control Groups (cgroup)**. CPU quota control how much CPU time a process in a container can use.

{% include container-definition.md %}

### CPU Quota is a absolute value

Whereas **cpu_shares** are a relative value, **cpu_quota** is the number of microseconds of CPU time a container can use per **cpu_period**.
For example configuring:
 * **cpu_quota** to 50,000 
 * **cpu_period** to 100,000
 
The container will be allocated 50,000 microseconds per 100,000 microsecond period. A bit like (see below) the use of 0.5 CPUs.
Quota can be greater than the period. For example:
 * **cpu_quota** to 200,000 
 * **cpu_period** to 100,000
 
Now the container can use 200,000 microseconds of CPU time every 100,000 microseconds. To use the CPU time there will either need
to be multiple processes in the container, or a multi-threaded process. This configuration is a bit like (see below) having 2 CPUs.

The default **cpu_period** on most platforms is 100,000 microseconds or 100 milliseconds.

### Once your time is up

After a container uses its **cpu_quota** it is **throttled** for the remainder of the **cpu_period**. 
A multi-threaded process, or a container with multiple processes can use its quota up in a shorter time than its quota.
Multiple threads or processes in the same container executing on different CPUs in parallel all use up the quota. For example:
 * **cpu_quota** to 200,000 
 * **cpu_period** to 100,000
 * Container with a process with 10 threads
 
 
<img src="/assets/cpu_quota/stop-watch.jpg" class="img-fluid mt-1 pl-5 pr-5" />
 
The 10 threads can use 200,000 microseconds of CPU quota on 10 different CPUs in 20,000 microseconds. Linux then throttles the container 
for the remaining 80,000 microseconds of the period.
Tail latency for multi-threaded containers that use **cpu_quota** are commonly high due to this throttling.

### Takeaways

**cpu_quota** allows setting an upper bound on the amount of CPU time a container gets. Linux enforces the limit even if CPU time
is available. Quotas can hinder utilisation while providing a predictable upper bounds on CPU time. 

{% include jd-course.html %}

