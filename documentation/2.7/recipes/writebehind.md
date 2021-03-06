---
---
# Ehcache Write-Behind

 

##Introduction
This page addresses the problem of database write overload and explains how the Ehcache Write-behind feature can be the solution.

## Problem

It's easy to understand how a cache can help reduce database loads and improve application performance in a read-mostly scenario. In read-mostly use-cases, every time the application needs to access data, instead of going to the database, data can be loaded from in-memory cache, which can be hundreds, or even thousands, of times faster than database.

However, for scenarios that require frequent updates to the stored data, to keep the data in cache and database in sync, every update to the cached data must invoke a simultaneous update to the database at the same time. Updates to the database are almost always slower, so this slows the effective update rate to the cache and thus the performance in general. When many write requests come in at the same time, the database can easily become a bottleneck or, even worse, be killed by heavy writes in a short period of time.

## Solution

The Write-behind feature provided by Ehcache allows quick cache writes with ensured consistency between cache data and database.

The idea is that when writing data into the cache, instead of writing the data into database at the same time, the write-behind cache saves the changed data into a queue and lets a backend thread to do the writing later. Therefore, the cache-write process can proceed without waiting for the database-write and, thus, be finished much faster. Any data that has been changed can be persisted into database eventually. In the mean time, any read from cache will still get the latest data.

A cache configured to perform asynchronous persistence, such as this, is called a Write-behind Cache.

There are many benefits of a Write-behind Cache. For example:

* Offload database writes
* Spread writes out to flatten peaks
* Consolidate multiple writes into fewer database writes

## Discussion

To implement a Write-Behind using Ehcache, one needs to register a CacheWriterFactory for Write-behind Cache and set the writeMode property of the cache to "write_behind".

CacheWriterFactory can create a writer for any data source(s), such as file, email, JMS or database. Typically, the database is the most common example of a data source.

Once a cache is configured as a Write-Behind cache, whenever a Cache.put is called to add or modify data, the cache will first update the cache data, just like a normal cache does, then it will save the change into a queue. A backend thread should be started when the cache is initialized and it will keep pulling data from the queue and it will call a Writer instance created by the CacheWriterFactory to persist the new data asynchronously.

In an un-clustered cache, the write-behind queue is stored in local memory. If the JVM dies, any data still in the queue will be lost.

In a clustered cache, the write-behind queue is managed by Terracotta Server Array. The background thread on each JVM will check the shared queue and save each data change left in the queue. With clustered Ehcache, this background process is scaled across the cluster for both performance and high availability reasons. If one client JVM were to go down, any changes it put into the write-behind queue can always be loaded by threads in other clustered JVMs, therefore will be applied to the database without any data loss.

There are many advanced configurations for Write-behind Cache. Because of the nature of asynchronous writing, there are also restrictions on when Write-Behind Cache can be used. For more information, see [write-through caching](/documentation/2.7/apis/write-through-caching).
