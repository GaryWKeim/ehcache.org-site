---
---

# Replication Overview

The following sections provide a documentation Table of Contents and additional information sources about replication.

## Distributed Caching With Terracotta

For the best distributed caching experience, see [Terracotta Server](http://terracotta.org/downloads/open-source/catalog), which is also open source and free to use.


## Other Replication Means Table of Contents

Note: these modules are generally outdated.

| Topic | Description |
|:-------|:------------|
|[RMI Replicated Caching](/documentation/2.8/replication/rmi-replicated-caching.html)|Ehcache provides replicated caching using RMI. To set up RMI replicated caching, you need to configure the CacheManager with a PeerProvider and a CacheManagerPeerListener. Then for each cache that will be replicated, you need to add one of the RMI cacheEventListener types to propagate messages. You can also optionally configure a cache to bootstrap from other caches in the cluster.|
|[JGroups Replicated Caching](/documentation/2.8/replication/jgroups-replicated-caching.html)|JGroups can be used as the underlying mechanism for the replication operations in Ehcache. JGroups offers a very flexible protocol stack, reliable unicast, and multicast message transmission. To set up replicated caching using JGroups, you need to configure a PeerProviderFactory. For each cache that will be replicated, you then need to add a cacheEventListenerFactory to propagate messages.|
|[JMS Replicated Caching](/documentation/2.8/replication/jms-replicated-caching.html)|JMS can also be used as the underlying mechanism for replication operations in Ehcache. The Ehcache jmsreplication module lets organisations with a message queue investment leverage it for caching. It provides replication between cache nodes using a replication topic, pushing of data directly to cache nodes from external topic publishers, and a JMSCacheLoader, which sends cache load requests to a queue.|



## Additional Information about Replication
The following pages provide additional information about replicated caching with Ehcache:

* [Replicated Caching FAQ](/documentation/2.8/faq.html#replicated-caching-faq)
* [Hibernate and Replicated Caching](/documentation/2.8/integrations/hibernate.html#configuring-replicated-caching-using-rmi-jgroups-or-jms)
