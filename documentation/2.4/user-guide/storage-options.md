---
---
# Storage Options <a name="Storage-Options"/>
 

Ehcache has three stores:

* a MemoryStore
* an OffHeapStore (BigMemory, Enterprise Ehcache only) and
* a DiskStore (two versions: open source and Ehcache Enterprise)

## Memory Store
The `MemoryStore` is always enabled. It is not directly manipulated, but is a component of every cache.

### Suitable Element Types
All Elements are suitable for placement in the MemoryStore. It has the following characteristics:

* **Safe**
  * Thread safe for use by multiple concurrent threads.
  * Tested for memory leaks. See MemoryCacheTest#testMemoryLeak. This test passes for Ehcache but exploits a number of memory leaks in JCS. JCS will give an OutOfMemory error with a default 64M in 10 seconds.
* **Backed By JDK LinkedHashMap**&mdash;The `MemoryStore` for JDK1.4 and JDK 5 it is backed by an extended [LinkedHashMap](http://java.sun.com/j2se/1.4.2/docs/api/). This provides a combined linked list and a hash map, and is ideally
suited for caching. Using this standard Java class simplifies the
implementation of the memory cache. It directly supports
obtaining the least recently used element.

* **Fast**&mdash;The memory store, being all in memory, is the fastest caching option.

### Memory Use, Spooling and Expiry Strategy

All caches specify their maximum in-memory size, in terms of the number of elements, at configuration time.

When an element is added to a cache and it goes beyond its maximum
memory size, an existing element is either deleted, if overflowToDisk
is false, or evaluated for spooling to disk, if overflowToDisk is true.

In the latter case, a check for expiry is carried out. If it is expired
it is deleted; if not it is spooled. The eviction of an item from
the memory store is based on the 'MemoryStoreEvictionPolicy' setting
specified in the configuration file.
 
`memoryStoreEvictionPolicy` is an optional attribute in ehcache.xml
introduced since 1.2. Legal values are LRU (default), LFU and FIFO.

LRU, LFU and FIFO eviction policies are supported. LRU is the default,
consistent with all earlier releases of ehcache.

* **Least Recently Used (LRU)** *Default**&mdash;The eldest element, is the Least Recently Used (LRU). The last used timestamp is updated when an element is put into the cache or an element is retrieved from the cache with a get call.
* **Least Frequently Used (LFU)**&mdash;For each get call on the element the number of hits is updated. When a put call is made for a new element (and assuming that the max limit is reached for the memory store) the element with least number of hits, the Less Frequently Used element, is evicted.
* **First In First Out (FIFO)**&mdash;Elements are evicted in the same order as they come in. When a put call is made for a new element (and assuming that the max limit is reached for the memory store) the element that was placed first (First-In) in the store is the candidate for eviction (First-Out).

For all the eviction policies there are also `putQuiet` and `getQuiet` methods which do not update the last used timestamp.

When there is a `get` or a `getQuiet` on an element, it is checked for expiry. If expired, it is removed and null is returned.

Note that at any point in time there will usually be some expired
elements in the cache. Memory sizing of an application must always take into
account the maximum size of each cache. There is a convenience method
which can provide an estimate of the size in bytes of the `MemoryStore`.

See [calculateInMemorySize()](/apidocs/net/sf/ehcache/Cache.html#calculateInMemorySize%28%29).
It returns the serialized size of the cache. Do not use this method in production. It is very slow. It is only meant to provide a rough estimate. The alternative would have been to have an expiry thread. This is a trade-off between lower memory use and short locking periods and cpu utilisation. The design is in favour of the latter. For those concerned with memory use, simply reduce the `maxElementsInMemory`.

## BigMemory (Off-Heap Store)

[BigMemory](http://www.terracotta.org/bigmemory?src=ehcache_off_heap_store) is a pure Java product from Terracotta that permits caches to use an additional
type of memory store outside the object heap.  It is packaged for use in Enterprise Ehcache as a snap-in job store called the "off-heap store."

This off-heap store, which is not subject to Java GC, is 100 times faster than the DiskStore and allows very large caches to be created (we have tested this up to 350GB).
Because off-heap data is stored in bytes, there are two implications:

* Only Serializable cache keys and values can be placed in the store, similar to DiskStore.
* Serialization and deserialization take place on putting and getting from the store. This means that the
 off-heap store is slower in an absolute sense (around 10 times slower than the MemoryStore), but this
 theoretical difference disappears due to two effects:
* the MemoryStore holds the hottest subset of data from the off-heap store, already in deserialized form
* when the GC involved with larger heaps is taken into account, the off-heap store is faster on average

### Suitable Element Types

Only `Element`s which are `Serializable` can be placed in the `OffHeapMemoryStore`. Any non serializable
`Element`s which attempt to overflow to the `OffHeapMemoryStore` will be removed instead, and a WARNING level
log message emitted.

See the [BigMemory](./bigmemory) chapter for more details.

## DiskStore <a name="DiskStore"/>

The `DiskStore` provides a disk spooling facility.

### DiskStores are Optional

The diskStore element in ehcache.xml is now optional (as of 1.5). If all caches use only `MemoryStore`s,
then there is no need to configure a diskStore. This simplifies configuration, and uses less threads. It is also good where where multiple CacheManagers are being used, and multiple disk store paths would need to be configured.

If one or more caches requires a DiskStore, and none is configured, java.io.tmpdir will be used and
a warning message will be logged to encourage explicity configuration of the diskStore path.

#### Turning off Disk Stores

To turn off disk store path creation, comment out the diskStore element in ehcache.xml.

The `ehcache-failsafe.xml` configuration uses a disk store. This will remain the case so as to not affect
existing Ehcache deployments. So, if you do not wish to use a disk store make
sure you specify your own ehcache.xml and comment out the diskStore element.

### Suitable Element Types

Only `Element`s which are `Serializable` can be placed in the DiskStore. Any non serializable
`Element`s which attempt to overflow to the `DiskStore` will be removed instead, and a WARNING level
log message emitted.

### Enterprise DiskStore

The commercial version of Ehcache 2.4 introduced an upgraded disk store. Improvements include:

* Upgraded fragmentation control/management to be the same as offheap
* No Heap used for fragmentation management or keys
* Much more predictable write latency up to caches over half a terabyte.
* SSD aware and optimised.

Throughput is approximately 110,000 operations/s which translates to around 60MB/sec on a 10k rpm  hard drive
with even higher rates on SSD drives, for which the Disk

### Storage

#### Files

The disk store creates a data file for each cache on startup called "<cache_name>.data". If the
 `DiskStore` is configured to be persistent, an index file called "**cache name**.index" is created on flushing
 of the `DiskStore` either explicitly using `Cache.flush` or on `CacheManager` shutdown.

#### Storage Location

 Files are created in the directory specified by the diskStore
configuration element. The diskStore configuration for the ehcache-failsafe.xml and bundled sample
configuration file ehcache.xml is "java.io.tmpdir", which causes files to be created in the system's temporary
directory.

#### &lt;diskStore&gt; Configuration Element
The `diskStore` element is has one attribute called `path`.

</pre>
&lt;diskStore path="java.io.tmpdir"/&gt;
</pre>

Legal values for the path attibute are legal file system paths.  E.g., for Unix:

<pre>
/home/application/cache
</pre>

The following system properties are also legal, in which case they are translated:

* user.home - User's home directory
* user.dir - User's current working directory
* java.io.tmpdir - Default temp file path
* ehcache.disk.store.dir - A system property you would normally specify on the command line&mdash;for example, java -Dehcache.disk.store.dir=/u01/myapp/diskdir ...

Subdirectories can be specified below the system property, for example:

<pre>
java.io.tmpdir/one
</pre>

becomes, on a Unix system:

<pre>
/tmp/one
</pre>

### Expiry
 One thread per cache is used to remove expired elements. The optional attribute `diskExpiryThreadIntervalSeconds`
sets the interval between runs of the expiry thread. Warning: setting this to a low value
is not recommended. It can cause excessive `DiskStore` locking and high cpu utilisation. The default value is 120 seconds.

### Eviction
 If the `maxElementsOnDisk` attribute is set, elements will be evicted from the `DiskStore` when it exceeds that amount.
The LFU algorithm is used for these evictions. It is not configurable to use another algorithm.

### Serializable Objects
 Only Serializable objects can be stored in a `DiskStore`. A [NotSerializableException](http://java.sun.com/j2se/1.4.2/docs/api/java/io/NotSerializableException.html)
will be thrown if the object is not serializable.

### Safety
 `DiskStore`s are thread safe.

### Persistence <a name="Persistence"/>
`DiskStore` persistence is controlled by the diskPersistent configuration element. If false or omitted, `DiskStore`s will not persist
between `CacheManager` restarts. The data file for each cache will be deleted, if it
exists, both on shutdown and startup. No data from a previous instance `CacheManager` is available.

 If diskPersistent is true, the data file, and an index file, are saved. Cache Elements are
available to a new `CacheManager`. This `CacheManager` may be in the same VM instance, or a new one.

The data file is updated continuously during operation of the Disk Store if `overflowToDisk` is true.
Otherwise it is not updated until either `cache.flush()` is called or the cache is disposed.

In all cases the index file is only written when dispose is called on the `DiskStore`. This happens when the CacheManager is shut down, a Cache is disposed,
or the VM is being shut down. It is recommended that the
CacheManager [shutdown()](http://ehcache.org/apidocs/net/sf/ehcache/CacheManager.html#shutdown%28%29)
method be used. See Virtual Machine Shutdown Considerations for guidance on how to safely shut the Virtual
Machine down.
 
When a `DiskStore` is persisted, the following steps take place:

* Any non-expired Elements of the `MemoryStore` are flushed to the DiskStore
* Elements awaiting spooling are spooled to the data file
* The free list and element list are serialized to the index file
 On startup the following steps take place:
* An attempt is made to read the index file. If it does not exist
or cannot be read successfully, due to disk corruption, upgrade of
ehcache, change in JDK version etc, then the data file is deleted and
the `DiskStore` starts with no Elements in it.
* If the index file is read successfully, the free list and element
list are loaded into memory. Once this is done, the index file contents
are removed. This way, if there is a dirty shutdown, when restarted,
Ehcache will delete the dirt index and data files.
* The `DiskStore` starts. All data is available.
* The expiry thread starts. It will delete Elements which have expired.
 These actions favour safety over persistence. Ehcache is a cache, not a
database. If a file gets dirty, all data is deleted. Once started there
is further checking for corruption. When a get is done, if the Element
cannot be successfully derserialized, it is deleted, and null is
returned. These measures prevent corrupt and inconsistent data being
returned.

* Fragmentation&mdash;Expiring an element frees its space on the file. This space is available for reuse by new
elements. The element is also removed from the in-memory index of elements.
* Serialization&mdash; Writes to and from the disk use [ObjectInputStream](http://java.sun.com/j2se/1.4.2/docs/api/java/io/ObjectOutputStream.html)
and the Java serialization mechanism. This is not required for the
MemoryStore. As a result the DiskStore can never be as fast as the MemoryStore.

 Serialization speed is affected by the size of the objects being
serialized and their type. It has been found in the ElementTest test
that:
  * The serialization time for a Java object being a large Map of String arrays was 126ms, where the a serialized size was 349,225 bytes.
  * The serialization time for a byte[] was 7ms, where the serialized size was 310,232 bytes

 Byte arrays are 20 times faster to serialize. Make use of byte arrays to increase DiskStore performance.

* RAMFS&mdash;One option to speed up disk stores is to use a RAM file system. On some operating systems there are a plethora of file systems to
choose from. For example, the Disk Cache has been successfully used
with Linux' RAMFS file system. This file system simply consists of
memory. Linux presents it as a file system. The Disk Cache treats it
like a normal disk - it is just way faster. With this type of file
system, object serialization becomes the limiting factor to performance.

  <ul>
  <li><p>Operation of a Cache where {overflowToDisk is false and diskPersistent} is true</p>

    <p>In this configuration case, the disk will be written on `flush` or `shutdown`.</p>
 
    <p>The next time the cache is started, the disk store will initialise but will not permit
    overflow from the MemoryStore. In all other respects it acts like a normal disk store.</p>
 
    <p>In practice this means that persistent in-memory cache will start up with all of its elements
    on disk. As gets cause cache hits, they will be loaded up into the `MemoryStore`. The oher
    thing that may happen is that the elements will expire, in which case the `DiskStore` expiry
    thread will reap them, (or they will get removed on a get if they are expired).</p>
 
    <p>So, the Ehcache design does not load them all into memory on start up, but lazily loads them as
    required.</p></li>
    </ul>

## Some Configuration Examples

These examples show how to allocate 8GB of machine memory to different stores. It assumes a
data set of 7GB - say for a cache of 7M items (each 1kb in size).

Those who want minimal application response time variance (ie minimizing GC pause times), will likely
want all the cache to be off-heap.
Assuming that 1GB of heap is needed for the rest of the app, they will set their Java config as follows:

<pre>
java -Xms1G -Xmx1G -XX:maxDirectMemorySize=7G
</pre>

And their Ehcache config as:

<pre>
&lt;cache
 maxElementsInMemory=100
 overflowToOffHeap="true"
 maxMemoryOffHeap="7G"
... /&gt;
</pre>

Those who want best possible performance for a hot set of data while still reducing overall application repsonse time variance will likely want a combination of on-heap and off-heap. The heap will be used for the hot set, the offheap for the rest. So, for example
if the hot set is 1M items (or 1GB) of the 7GB data. They will set their Java config as follows

<pre>
java -Xms2G -Xmx2G -XX:maxDirectMemorySize=6G
</pre>

And their Ehcache config as:

<pre>
&lt;cache
  maxElementsInMemory=1M
  overflowToOffHeap="true"
  maxMemoryOffHeap="6G"
  ...&gt;
</pre>

This configuration will compare VERY favorably against the alternative of keeping the less-hot set in a
database (100x slower) or caching on local disk (20x slower).

Where pauses are not a problem, the whole data set can be kept on heap:

<pre>
&lt;cache
  maxElementsInMemory=1
  overflowToOffHeap="false"
  ...&gt;
</pre>

Where latency isn't an issue overflow to disk can be used:

<pre>
&lt;cache
  maxElementsInMemory=1M
  overflowToOffDisk="true"
  ...&gt;
</pre>

## Performance Considerations

### Relative Speeds

Ehcache comes with a `MemoryStore` and a `DiskStore`. The `MemoryStore` is approximately
an order of magnitude faster than the `DiskStore`. The reason is that
the `DiskStore` incurs the following extra overhead:

* Serialization of the key and value
* Eviction from the `MemoryStore` using an eviction algorithm
* Reading from disk

Note that writing to disk is not a synchronous performance overhead
because it is handled by a separate thread.

### Always use some amount of Heap

A Cache should alway have its `maximumSize` attribute
set to 1 or higher. A Cache with a maximum
size of 1 has twice the performance of a disk only cache, i.e. one
where the `maximumSize` is set to 0. For this reason a warning will be issued if a Cache is
created with a 0 `maximumSize`.

And when using the Offheap Store, frequently accessed elements can be held in heap in derserialized form
if an Onheap (configured with maxElementsInMemory) store is used