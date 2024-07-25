---
layout: page
permalink: /platforms/
title: Platform Engineering
---

Most organisations, perhaps under a different name, are building a Platform as a Service (PaaS).
Platforms range from the fully third-party managed to the fully built in-house. The former offers
the quickest time to market whereas the latter gives ultimate flexibility as your business evolves.
Typically, your organisation wants to pick somewhere in between.

## What is a Platform?

A platform provides the services necessary to run an application in production. Service teams bundle their 
application in a way defined by the platform e.g. Docker images, and the platform runs it. 

The most complete platform includes:

* [Ingress](./systems.md#system-ingress) including load balancing, TLS termination
* [Persistence](./systems.md#persistence-technology)
* Service discovery for [inter process communication](./systems.md#inter-process-communication)
* Observability, including the interface for service teams to push metrics and traces
* Monitoring including alerts

In some cases full control over the path to production:

* Continuous integration
* Continuous deployment

In short a platform provides everything a [system](./systems.md) needs to run.
In our experience even the most complete third-party PaaS leaves a lot of decisions up the organisation:

* Environments e.g. Dev, UAT, Stage, Production
* Production testing: how to implement or use Canarying or Blue-Green deployments
* Standardising observability and monitoring

## Cloud != Platform

Using a cloud provider such as AWS or GCP doesn't mean you don't need a platform. Even if you use their most
managed features such as [Google App Engine](https://cloud.google.com/appengine) you as least need to decide
how to implement many of the parts of your platform with the Cloud offering.




