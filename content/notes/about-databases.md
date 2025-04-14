---
layout: post
title:  "Databases"
description: "Types of databases, how they are scaled and what to choose when"
categories: ["system-design"]
date: 2024-03-15 19:45:31 +0530
author: "Sai Kiran"
comments: false
---

### SQL vs NoSQL

SQL databases have been there for a long time, and they assume that data reside in single node or same machine. That is their trait, and they provide ACID compliance. Later databases were made distributed so that we will get away with vertical hardware scaling limitations and create database that can store huge amount of data by leverating multiple machines/nodes. Most of the NoSQL databases are bydefault distributed, hence they provide eventual consistency. Even if we make some SQL database distributed then we will have to give up on ACID compliance, unless you want to have synchronous replication setup and are ready to comprimise on the throughput.

[ SQL VS NO-SQL | SIMILARITIES AND DIFFERENCES | Podcast with Arpit Bhayani](https://www.youtube.com/watch?v=4yI9qx0NG9A) has good insights into this topic.
Also, refer his tweets.
https://x.com/arpit_bhayani/status/1705423114488955333
https://x.com/arpit_bhayani/status/1727913152875241769

Got good insights on replication, sharding, indexing etc at [Arpit's blog](https://arpitbhayani.me/knowledge-base/database-engineering). [Consistent Hashing](https://arpitbhayani.me/blogs/consistent-hashing) is also explained very well here, which is used to add more nodes to database cluster that maybe shared also which incurring much data movement.

In the above podcast as Arpit discussed, we need to first identify what we need and what can compromise, based on that we can choose the database. 
SQL databases provide strong consistency but they assume the data resides in single node. Synchronous replication helps with strong consistency with distributed SQL nodes but compromises on the throughput. Looks like CockroachDB is ACID compliant while being distributed.

When data is distributed, then consistency will be not easy to achieve, we might have to compromize on certain things.

If the amount of data can fit in one node, SQL is simple. If data is humungous and can't fit in single node and we can be fine with eventual consistency then go for NOSQL variants. 

But as long as business needs are satisfied no need to introduce fancy technology, if one node of SQL databases can run your business you are good!