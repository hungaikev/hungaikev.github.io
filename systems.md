---
layout: page
permalink: /systems/
title: System Architecture
---

What is a `system`? A system fulfils a business need. A `system` is made up of:

* Processes a.k.a services
* Databases e.g. Apache Cassandra
* Queueing systems e.g. Apache Kafka   

Users needn't know about these components, only the `system`'s `ingress`. Users interact with your system via its `ingress`. 
Typical `ingress` mechanisms are:

* HTTP and HTTP/2
* gRPC over HTTP/2
* A topic/queue such as Apache Kafka

Users can be humans via a web browser or other `systems`.

What is system architecture? Simply put, architecture consists of everything that is hard to change after the fact.
Your system architecture impacts:

* Reliability: does your system give the user the correct answer
* Availability: can your system respond to user requests
* Performance: how quickly does your system respond 
* Capacity:  how many concurrent requests can your system handle

Our recommended approach is to define [Service Level Objectives (SLOs)](https://sre.google/sre-book/service-level-objectives/) for these aspects of your system even if you don't
 a [Service Level Agreement (SLA)](https://en.wikipedia.org/wiki/Service-level_agreement) with your user.

Some important parts of system architecture that constrain the above are:

* System ingress
* Interprocess communication 
* Persistence technologies e.g. databases & queues
* Database access patterns 

### System ingress

A user interacts with a `system` via its `ingress`. There are different tradeoffs for `ingress` vs communication within a system.
For `ingress` typical concerns are:

* Forward and backward compatibility: users are out of your control, within a system all the components are in your control
* Authentication and authorisation
* TLS
* Global access: is the system exposed to the web, should their be edge locations around the world or a single region?

For externally accessible systems the most common ingress technology is HTTP. There are many benefits to sticking with HTTP:

* Global HTTP ingress support cloud providers with support for TLS termination
* Contend Delivery Networks and other caching technology
* Well defined authentication mechanisms and single sign on (SSO)

For internal systems other ingress technologies can be used that are typical for inter process communication with in a system:

* Queues e.g. Kafka: Asynchronous ingress allows the system to be fully shut down for maintenance without users knowing as long as performance SLOs aren't breached
* HTTP/2 can take advantage of TCP multiplexing to reduce the overhead of connections
* gRPC uses, and has the advantages of, HTTP/2 as well as defining interfaces and serializion formats

### Inter process communication

The first major decision for interprocess communication is whether it is asynchronous or synchronous. Queues such as Kafka allow
asynchronous communication allowing components to be fully shutdown at the expense of two additional complexities:

* Communication complexity for request/reply 
* Infrastructure complexity of running queueing technology

Synchronous communication means that the downstream component needs to be up at the time of request. It allows easier request/reply communication.
Synchronous communication is susceptible to cascading failures, where a downstream failure causes failures in all upstream components.

### Database access patterns

Database access patterns come in three main categories:

* Create, Update, Read, Delete (CRUD) where the database stores the current state of entities such as users, transactions or stock
* Arbitrary access. Less principled than CRUD, using any/every feature of a database e.g. arbitrary SQL.
* Event sourcing and Command Query Read Segregation (CQRS)

CRUD often degrades into arbitrary database access. Event sourcing opens up possibilities such as rebuilding different views on the data
as all historical events are stored but introduces complexity such as:

* Large build up of events, slowing the system down
* Supporting old schemas for events

Each process in a system can use different database access patterns. Some organisations will give full autonomy to development teams to decide
on internal data access patterns whereas others will want to standardise. 

## Persistence technology

Nothing affects a system more than the selected persistence technology. It puts upper bounds on:

* Capacity: A traditional relational database may hit its limit in the 10s of TBs whereas a distributed database such as Apache Cassandra can go up to Petabytes
* Availability: Can it be deployed across DCs or is the max availability that of a single cloud region or data center?
* Performance

In addition, it will dictate the costs of a system. Using [Google Spanner](https://cloud.google.com/spanner) may provide your system with excellent latency and multi region support
but it comes at a huge cost.



