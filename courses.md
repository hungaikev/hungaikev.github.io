---
layout: page
permalink: /courses/
title: Courses
---
   
I'm available for doing online and in person training. 
If you're interested in arranging please [get in touch](mailto:hungaikevin@gmail.com).

### The complete guide to running JVM applications inside Docker/Kubernetes

If you need to learn how to run, tune, and maintain JVM applications
that run in Docker and/or Kubernetes then this is the course for you.

This course is very different from other Java/Docker/Kubernetes courses.
It focuses on all the skills that you need to succeed in production.

We'll start with introductions for Docker and Kubernetes then we'll get into the fun stuff. We'll learn:
- What a container is under the covers
- Linux cgroups
- Linux namespaces

Then we'll go into how the JVM and your Java application behave differently
in Kubernetes when running inside cgroups and namespaces. We'll cover:
- JVM ergonomics
- How CPU Shares and Quota work
- How Kubernetes manages CPU and Memory 

Then we'll teach you all the techniques needed to build production ready images:
- Selecting a base image
- JDK vs JRE based images
- Multi stage Docker builds
- GraalVM
- Class data sharing

All while experimenting with different JVM versions and settings.

By the end of this course you'll know how to:
- Build a production ready image
- Select between using CPU limits, quotas, or both in Kubernetes
- Select memory limits and tune the JVM for running in Kubernetes
- Understand CPU usage in Kubernetes and know why it is different to VMs and physical machines.
