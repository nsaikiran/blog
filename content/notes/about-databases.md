---
layout: post
title:  "Databases"
description: "Types of databases, how they are scaled and what to choose when"
categories: ["system-design"]
date: 2025-04-14 19:45:31 +0530
author: "Sai Kiran"
comments: false
---

### SQL vs NoSQL

SQL databases have been there for a long time, and they assume that data reside in single node or same machine. That is their trait, and they provide ACID compliance. Later databases were made distributed so that we will get away with vertical hardware scaling limitations and create database that can store huge amount of data by leverating multiple machines/nodes. Most of the NoSQL databases are bydefault distributed, hence they provide eventual consistency. Even if we make some SQL database distributed then we will have to give up on ACID compliance, unless you want to have synchronous replication setup and are ready to comprimise on the throughput.

[ SQL VS NO-SQL | SIMILARITIES AND DIFFERENCES | Podcast with Arpit Bhayani](https://www.youtube.com/watch?v=4yI9qx0NG9A) has good insights into this topic.
Also, refer his tweets.
https://x.com/arpit_bhayani/status/1705423114488955333
https://x.com/arpit_bhayani/status/1727913152875241769

Got good insights on replication, sharding, indexing etc at [Arpit's blog](https://arpitbhayani.me/blogs). 

In the above podcast as Arpit discussed, we need to first identify what we need and what can compromise, based on that we can choose the database.
SQL databases provide strong consistency but they assume the data resides in single node. Synchronous replication helps with strong consistency with distributed SQL nodes but compromises on the throughput. Looks like CockroachDB is ACID compliant while being distributed.

When data is distributed, then consistency will be not easy to achieve, we might have to compromize on certain things.

If the amount of data can fit in one node, SQL is simple. If data is humungous and can't fit in single node and we can be fine with eventual consistency then go for NOSQL variants. 

But as long as business needs are satisfied no need to introduce fancy technology, if one node of SQL databases can run your business you are good!

#### To improve reads in the system

- [CQRS](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs): Segregate reads and writes.
- [Materialized View pattern.](https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view): Prepopulate views to power reads
- [Data partitioning](https://learn.microsoft.com/en-us/azure/architecture/best-practices/data-partitioning): Partition data to improve performance.
- [Event sourcing](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)

There is some good information on Microsoft's architecture center, like the above, refer those.

[Consistent Hashing](https://arpitbhayani.me/blogs/consistent-hashing) is also explained very well here, which is used to add more nodes to database cluster that maybe shared also which incurring much data movement.

### Distributed systems

When data doesn't fit in single node then we will go for distributed data, also we need replicas for redundancy sake and to achieve CQRS or other optimizations. When we introduce replicas, we need to make sure they are in sync. Now, when we have setup replication. 

Now there will be one node identified as leader which takes all writes and others follow through replication. Now to reduce chances of data loss in case if leader dies before replication is done, leader will wait till some quorum of followers acknowledged replication (synchronous replication, but it increases write operation time). That way the chances of data loss will be less, usually in one region, leader will be in one availability zone and it will wait for quorum of followers from different availability zones to confirm. This saves us from failure of one availability zone, but what if we want to handle whole region failures, then we enable replication between regions.

We are trying to prevent the data loss. If we do synchronous replication then writes *will take time*, if we keep asynchronous replication then chances of data loss will be there.

Refer ChatGPT conversation [here](https://chatgpt.com/share/6902ebbf-1e0c-8008-bca0-b3b351655682) about this topic.
Refer Arpit's [blog](https://arpitbhayani.me/blogs), he gave good explanations about various methods used in industry in distributed databases, and more.