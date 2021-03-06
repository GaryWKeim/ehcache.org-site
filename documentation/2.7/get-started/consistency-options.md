---
---
# Cache Consistency Options

 


## Introduction
Consistency is a quality of shared data that is relevant for distributed caching. This page explains the distributed-cache consistency models in terms of standard distributed systems theory. Distributed caches are available when using the Terracotta Server Array, available with [BigMemory Max](http://terracotta.org/products/bigmemorymax).


## Propagation of Modified Data
In general, any data cached in the heap is always consistent in that heap. For example, if one thread gets and modifies a cached element, another thread that then gets that element sees the change as long as all activity takes place in that heap.

If, however, other data tiers are involved, it is possible for the second thread to get the original element, not the changed element. If a `cache.get("key").getValue().modify()` happens, after which the element is flushed to a lower tier (to BigMemory or disk, for example), then a subsequent get obtains the original element (not the changed element). To ensure that subsequent `get()` operations obtain the changed element, the changed element should be put back into the cache. To guarantee this behavior without having to perform a `put()`, set the cache's `copyOnRead` attribute to false.

## Server-Side Consistency
Leaving aside the issue of data also held in the Ehcache nodes, let us look at the server side consistency of the Terracotta Server Array (TSA).

### Server Deployment Topology
Large datasets are handled with partitions which are managed automatically using a consistent
hashing algorithm once a set of "stripes" are defined in the &lt;tc-config>. There is no dynamic resizing of clusters, so the consistent hash always resolves to the same stripe.
The TSA is typically deployed with a pair of servers per partition of data, which is known in the &lt;tc-config> as a Mirror Group.
A mirror group has an active server which handles all requests for that partition and a mirror server, or hot standby, which does not
service any requests. The active server propagates changes to the mirror server.
In the language of consistency protocols, the active and mirror are replicas - they should contain the same data.


### Restating in terms of Quorum based replicated-write protocols
To use the terminology from Gifford (1979), a storage system has *N* storage replicas. A write is a *W*. A read is an *R*.
The server-side storage system will be strongly consistent if:

* R + W > N

    and

* W > N/2

In Terracotta, there is one active and one mirror. The acknowledgement is not sent until all have been written to. We always read from only one replica, the Active. So R = 1, W = 2, N = 2.
Substituting the terms of R + W > N, we get 1 + 2 > 2, which is clearly true. And for W > N/2 we get 2 > 2/2 => 2 > 1 which is clearly true. Therefore we are strongly consistent server side.


## Client-Side Consistency
Because data is also held in Ehcache nodes, and Ehcache nodes are what application code interacts with, there is more to the story than consistency in the TSA. Werner Vogel's seminal "Eventually Consistent" paper presented standard terms for client-side consistency and a way of reasoning about whether that consistency can be achieved in a distributed system. This paper in turn referenced Tannenbaum's [Distributed Systems: Principles and Paradigms (2nd Edition)](http://www.amazon.com/Distributed-Systems-Principles-Paradigms-2nd/dp/0132392275/ref=dp_ob_title_bk).

Tannenbaum was popularising research work done on Bayou, a database system. See page 290 of "Distributed Systems, Principles and Paradigms" by Tannenbaum and Van Steen for detailed coverage of this material.

### Model Components
Before explaining our consistency modes, we need to explain the standard components of the reference model, which is an abstract model of a distributed system that can be used for studying interactions.

* A storage system. The storage system consists of data stored durably in one server or multiple servers connected via a network. In Ehcache, durability is optional and the storage system might simply be in memory.
* Client Process A. This is a process that writes to and reads from the storage system.
* Client Processes B and C. These two processes are independent of process A and write to and read from the storage system. It is irrelevant whether these are really processes or threads within the same process; what is important is that they are independent and need to communicate to share information. Client-side consistency has to do with how and when observers (in this case the processes A, B, or C) see updates made to a data object in the storage systems.

### Mapping the Model to Distributed Ehcache
The model maps to Distributed Ehcache as follows:

* there is a Terracotta Server Array which is the 'storage system';
* there are three nodes connected to the Terracotta Server Array: Ehcache A, B and C, mapping to the processes in the standard model;
* a "write" in the standard model is a "put" or "remove" in Ehcache.

### Standard Client-Side Consistency Modes
It then goes on to define the following consistencies where process A has made an update to a data object:

* Strong consistency. After the update completes, any subsequent access (by A, B, or C) will return the updated value.
* Weak consistency. The system does not guarantee that subsequent accesses will return the updated value.
* Eventual consistency. This is a specific form of weak consistency; the storage system guarantees that if no new updates are made to the object, eventually all accesses will return the last updated value. If no failures occur, the maximum size of the inconsistency window can be determined based on factors such as communication delays, the load on the system, and the number of replicas involved in the replication scheme.

Within eventual consistency there are a number of desirable properties:

* Read-your-writes consistency. This is an important model where process A, after it has updated a data item, always accesses the updated value and will never see an older value. This is a special case of the causal consistency model.
* Session consistency. This is a practical version of the previous model, where a process accesses the storage system in the context of a session. As long as the session exists, the system guarantees read-your-writes consistency. If the session terminates because of a certain failure scenario, a new session needs to be created and the guarantees do not overlap the sessions.
* Monotonic read consistency. If a process has seen a particular value for the object, any subsequent accesses will never return any previous values.
* Monotonic write consistency. In this case the system guarantees to serialize the writes by the same process. Systems that do not guarantee this level of consistency are notoriously hard to program.

Finally, in eventual consistency, the period between the update and the moment when it is guaranteed that any observer will always see the updated value is dubbed the inconsistency window.

## Consistency Modes in Distributed Ehcache
The consistency modes in Terracotta distributed Ehcache are "strong" and "eventual". Eventual consistency is the default mode.

### Strong Consistency
In the distributed cache, strong consistency is configured as follows:

    <cache name="sampleCache1"
    ...
     />
       <terracotta consistency="strong" />
    </cache>

We will walk through how a write is done and show that it is strongly consistent.

1.    A thread in Ehcache A performs a write.
2.    Before the write is done, a write lock is obtained from the Terracotta Server (storage system).
          The write lock is granted only after all read locks have been surrendered.
3.    The write is done to an in-process Transaction Buffer. Within the Java process the write is thread-safe.
          Any local threads in Ehcache A will have immediate visibility of the change.
4.    Once the change has hit the Transaction Buffer which is a LinkedBlockingQueue, a notify occurs, and the
          Transaction Buffer initiates sending the write (update) asynchronously to the TSA (storage system).
5.    The Terracotta Server is generally configured with multiple replicas forming a Mirror Group. Within the mirror group there is an active server, and one or more mirror servers.  The write is to the active server. The active server does not acknowledge the write until it has written it to each of the mirror servers in the mirror group.
           It then sends back an acknowledgement to Ehcache A which then deletes the write from the Transaction Buffer.
6.    A read or write request from Ehcache A is immediately available because a read lock is automatically granted when a write lock
          has already been acquired. A read or write request in Ehcache B or C requires the acquisition of a read or write lock,
          respectively, which will block until step 5 has occurred. In addition, if you have a stale copy locally, it is updated first.
          When the lock is granted, the write is present in all replicas. Because Ehcache also maintains copies of Elements in-process
          in potentially each node, if any of Ehcache A, B or C have a copy they are also updated before Step 5 completes.

Note: This analysis assumes that if the `nonstop` is being used, it is configured with the default of Exception, so that on a `clusterOffline` event no cache operations happen locally. Nonstop allows fine-grained tradeoffs to be made in the event of a network partition, including dropping consistency.

### Eventual Consistency
Distributed Ehcache can have eventual consistency in the following ways:

* Configured with `consistency="eventual"`.
* Set programmatically with a bulk-loading mode, using `setNodeBulkLoadEnabled(boolean)`.
* Configured with &lt;UnlockedReadsView>, a `CacheDecorator` that can be created like a view on a cache to show the latest writes visible to the local Ehcache node without respect for any locks.
* Using bulk-loading Cache methods `putAll()`, `getAll()`, and `removeAll()`. (Note that `putAll(Collection<Element>)` does not discriminate between new and existing elements, thus resulting in put notifications, not update notifications.) These can also be used with strong consistency. If you can use them, it's unnecessary to use bulk-load mode. See the [API documentation](http://ehcache.org/apidocs) for details.

Regardless, Ehcache B and C will eventually see the change made by Ehcache A, generally with consistency window of 5ms (with no partitions or interruptions). If a GC happens on a TSA node, or Ehcache A or B, the inconsistency window is increased by the length of the GC. 

If `setNodeBulkLoadEnabled(true)` is used, it causes the TSA to not update Ehcache B and C. Instead, they are set to a 5 minute fixed TTL. The inconsistency window thus increases to 5 minutes plus the above.

If a network partition occurs that is long enough to cause an Ehcache A to be ejected from the cluster, the only configurable option is to discard on [rejoin](http://terracotta.org/documentation/2.7/4.0/bigmemorymax/configuration/reference-guide#71266). Once this happens Ehcache A or B gets the write. From the perspective of other threads in Ehcache A, all writes are thread-safe.

#### Java Memory Model Honored
In all modes the *happens-before* requirement of the Java Memory Model is honored. As a result the following is true:

*   A thread in Ehcache A will see any writes made by another thread. => Read your writes consistency.
*   Monotonic Read Consistency in Ehcache A is true.
*   Monotonic Write Consistency is Ehcache A is true.

#### Consistency in Web Sessions
It should be noted that desirable characteristics of eventual consistency are from the point of view of Ehcache A. From the context of a web application, in order for an end user interacting with a whole application to see this behaviour use sticky sessions.

This way the user interacts with the same node (Ehcache A) for each step. If an application node falls over, a new session will be established. The time between the last write, failure, detection by the load balancer and allocation to a new application node will take longer than the 5ms+ that it takes for all Ehcache nodes in the cluster to get the write. So when the new application node is switched to, eventual consistency has occurred and no loss of consistency is observed by the user.

If you want to avoid sticky sessions, try relying on the time gap between a click or submit and the next one in a "click path" that takes much longer than the 5ms+ that it takes for other nodes to become eventually consistent.

In an Internet context the user is sufficiently distant from the server so that the response time is at least an order of magnitude greater than the inconsistency window. Probabilistically it is therefore unlikely that a user would see inconsistency.

## Other Safety Features
Ehcache offers a rich set of data safety features. In this section we look at some of the others and how they interact with the
`strong` and `eventual` consistency.

### Compare and Swap Cache Operations
We support three Compare and Swap (CAS) operations:

*   `cache.replace(Element old, Element new)`
*   `cache.putIfAbsent(Element)`
*   `cache.removeElement(Element)`

In each case the TSA will only perform the write if the old value is the same as that presented. This is guaranteed to be done atomically as required by the CAS pattern.
CAS achieves strong consistency between A, B, and C with __optimistic locking__ rather than pessimistic locking. Optimistic locking approaches are not guaranteed to succeed. If someone else got in and changed the Element ahead of you, the methods will return `false`. You should read the new value, take that into account in your business logic, and then retry your mutation.

CAS will work with both `strong` and `eventual` consistency modes, but because it does not use the locks, it does not need `strong`. However, with eventual consistency two simultaneous `replace()` operations in different nodes (or threads) can both return true. But at the TSA, only one of the operations is allowed to succeed and all competing values are invalidated, eventually making the caches consistent in all nodes.


## Use Cases And Recommended Practices
In this section we look at some common use cases and give advice on what consistency and safety options should be used. These
serve as a useful starting point for your own analysis.
We welcome commentary and further discussion on these use cases. Please post to the Ehcache forums.

### Shopping Cart - optimistic inventory

#### Problem
A user adds items to a shopping cart. Do not decrement inventory until checkout.

#### Solution
Use eventual consistency.

### Shopping Cart with Inventory Decrementing

#### Problem
A user adds items to a shopping cart. There is limited inventory and the business policy is that the first user to add the inventory
to their shopping cart can buy it. If the user does not proceed to checkout, a timer will release the inventory back. As a result,
inventory must be decremented at the time the item is added to the shopping cart.

#### Solution
Use strong consistency with one of:

* explicit locking
* local transactions
* XA transactions

The key thing here is that two resources have to be updated: the shopping cart, which is only visible to one user, and on it's own
has low consistency requirements, and an inventory which is transactiional in nature.

### Financial Order Processing - write to cache and database

#### Problem
An order processing system sends a series of messages in a workflow, perhaps using Business Process Management software. The system
involves multiple servers and the next step in the processing of an order may occur on any server. Let's say there are 5 steps in the
process.
To avoid continual re-reading from a database, the processing results are also written to a distributed cache. The next step could execute
in a few ms to minutes depending on what other orders are going through and how busy the hardware is.

#### Solution
Use strong consistency plus XA transactions.
Because the execution step cannot be replayed once completed, and may be under the control of a BPM, it is very important that the
change in state gets to the cache cluster. Synchronous writes can also be used (at a high performance cost) so that the put to the cache does not
return until the data has been applied. If an executing node failed before the data was transferred, the locks would still be in place
preventing readers from reading stale data, but that will not help the next step in the process.
XA transactions are needed because we want to keep the database and the cache in sync.

### Immutable Data

#### Problem
The application uses data that once it comes into existence is immutable. Nothing is immutable forever. The key point is that it is
immutable up until the time of the next software release.
Some examples are:

*   application constants
*   reference data - zip and post codes, countries etc.

If you analyse database traffic commonly used reference data turns out to be a big hitter.
As they are immutable they can only be appended or read, never updated.

#### Solution
In concurrent programming, immutable data never needs further concurrency protection. So we simply want to use the fastest mode.
Here we would always use eventual consistency.

### Financial Order Processing - write to cache as SOR

#### Problem
An order processing system sends a series of messages in a workflow, perhaps using Business Process Management software. The system
involves multiple servers and the next step in the processing of an order may occur on any server. Let's say there are 50 steps in the
process.
To avoid overloading a database the processing results at each step only written to a distributed cache. The next step could execute
in a few ms to minutes depending on what other orders are going through and how busy the hardware is.

#### Solution
Use one of:

*  strong consistency and local transactions (if changes are needed to be applied to multiple caches or entries).
  Because the execution step, once completed cannot be replayed, and may be under the control of a BPM, it is very important that the
change in state gets to the cache cluster. Synchronous writes can also be used (at a high performance cost) so that the put to the cache does not
return until the data has been applied. If an executing node failed before the data was transferred, the locks would still be in place
preventing readers from reading stale data, but that will not help the next step in the process.
*  CAS operations with eventual consistency. The CAS methods will not return until the data has been applied to the server, so
  it is not necessary to use synchronous writes.
In a 50 step process it is likely there are key milestones. Often it is desirable to record these in a database with the non-milestone
   steps recorded in the cache. For these key milestones use the "Financial Order Processing - write to cache and database" pattern.

### E-commerce web app with Non-sticky sessions
Here a user makes reads and writes to a web application cluster. There are n servers where n > 1. The load balancer is non-sticky,
so any of the n servers can be hit on the next HTTP operation.
When a user submits using a HTML form, either a GET or POST is done based on the form action. And if it is an AJAx app, then
requests are being done with `XMLHttpRequest` and any HTTP request method can be sent. If POST (form and AJAX) or PUT (AJAX) is used,
no content is returned and a separate GET is required to refresh the view or AJAX app. 

The key point is that sending a change and getting
a view may happen with one request or two. If it happens with two, then the same server might respond to the second request or not.
The probability that the second server will be the same as the first is 1/n.
AJAX apps can further exacebate this situation. A page may make multiple requests to fill different panels. This opens up the possibility
of, within a single page, having data come from multiple servers. Any lack of consistency could be glaring indeed.

#### Solution
Use one of:

*   strong consistency
*   CAS

Other options can be added depending on what is needed for the request. e.g. XA if a database plus the cache is updated.

### E-commerce web app with sticky sessions

#### Problem
Here a user makes reads and writes to a web application cluster. The load balancer is sticky,
so the same server should be hit on the next HTTP operation. There are different ways of configuring sticky sessions. The same
server might be used for the length of a session, which is the standard meaning, or a browser's IP can permanently hash to a server.
In any case, each request is guaranteed to hit the same server.

#### Solution
The same server is always hit. The consistency mode depends on whether only the user making the changes needs to see them applied
(read your writes, monotonic reads, monotonic writes), or whether they are mutating shared-state, like inventory where write-write
conflicts might occur.
For mutating user-only consistency, use eventual consistency.
For multi-user shared state, use strong consistency at a minimum plus further safety mechanisms depending on the type of mutation.

### E-commerce Catalog

#### Problem
Catalogues display inventory. There are product details and pricing. There may be also be an inventory status of available or sold
out.
Catalogue changes are usually made by one user or process (for example a daily update load from a supplier) and usually do not
have write-write conflicts. While the catalogue is often non-sticky, admin users are typically configured sticky.
There is often tolerance for the displayed catalogue to lag behind the change made. Users following a click path are usually less tolerant about seeing inconsistencies.

#### Solution
The person making the changes can see a consistent view by virtue of the sticky session. So eventual consistency will often
be enough.
For end users following a click path, they need a consistent view. However, the network or Internet time, plus their think time to
move along the path, adds up to seconds and minutes, while eventual consistency will propagate in the order of 2+ milliseconds. With
eventual consistency, it is very unlikely they will see inconsistency. The general recommendation is therefore to use eventual consistency.
