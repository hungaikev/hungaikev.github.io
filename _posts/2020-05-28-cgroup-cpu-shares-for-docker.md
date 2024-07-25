---
layout: post
title: 'CPU Shares for Docker containers'
author: Christopher Batey
comments: true
date: '2020-05-28'
tags:
- docker 
- kubernetes
- containers
- cpu_shares
- cgroups
---

In this article we'll explain CPU shares, so you can understand how to set them in Docker.

**CPU shares (cpu_share)** are a feature of **Linux Control Groups (cgroup)**. CPU shares control how much CPU time a process in a container can use.

{% include container-definition.md %}
 
## CPU Shares are relative
 
A container's **cpu_share** is a relative value used for scheduling CPU time between different containers.
The **cpu_share** has no meaning in isolation. For example, setting a **cpu_share** of **512** gives you no information about
how much CPU time the container will get. Setting one container's **cpu_share** to **512** and another container's to **1024** 
means that the second container will get double the amount of CPU time as the first. That said, it still gives no information 
about the actual amount of CPU time each container will get, only the relative amount between them.

For example, if we have three containers on a machine with 1 CPU.
1. 1024 shares 
1. 512 shares 
1. 512 shares 

<img src="/assets/cpu_shares/one-cpu.png" class="img-fluid mt-1 pl-5 pr-5" />

Then each container will get:
1. 0.5 CPU
1. 0.25 CPU
1. 0.25 CPU

If the same three containers deployed to a machine with 2 CPUs.

<img src="/assets/cpu_shares/two-cpu.png" class="img-fluid mt-1 pl-5 pr-5" />

 Then each container will get:
 1. 1 CPU
 1. 0.5 CPU
 1. 0.5 CPU
 
 The key takeaway is that **cpu_shares** are relative, they give you no information without knowing what other containers
 are running on the machine and the machine's resources.
 
## CPU shares enforcement occurs during CPU contention

The above figures for how much CPU time a container has access to are only valid if every container 
wants to execute at the same time.

If only one container is active it can use all the CPU. If more than one container executes at once then the relative **cpu_shares**
dictate how much CPU time each container has access to. The **cpu_shares** of inactive containers are irrelevant.

The key take away is that **cpu_shares** allow for high utilisation. Any container, regardless of its cpu_shares, can use a machine's 
CPU resources if other containers are inactive. During CPU contention **cpu_shares** specify how much
CPU time each container gets.

## CPU Shares in Docker

Docker uses a base value of 1024 for **cpu_shares**. To give a container relatively more CPU set `--cpu-shares` for Docker run to a value greater than 1024. 
To give a container relatively less CPU time set `--cpu-shares` to lower than 1024.

If `--cpus` or `--cpu-quota` is set then even if there is no contention for CPU time the container will be limited based. This can be good for predictable 
resource usage but is generally bad for utilisation. 

{% include jd-course.html %}

