---
---
# Cache Decorators

 

## Introduction
Ehcache 1.2 introduced the Ehcache interface, of which Cache is an implementation. It is possible and encouraged
to create Ehcache decorators that are backed by a Cache instance, implement Ehcache and provide extra functionality.

The Decorator pattern is one of the the well known Gang of Four patterns.

Decorated caches are accessed from the CacheManager using `CacheManager.getEhcache(String name)`.
Note that, for backward compatibility, `CacheManager.getCache(String name)` has been retained. However only
`CacheManager.getEhcache(String name)` returns the decorated cache.

## Creating a Decorator

### Programmatically

Cache decorators are created as follows:

~~~ java
BlockingCache newBlockingCache = new BlockingCache(cache);
~~~

The class must implement Ehcache.

### By Configuration

Cache decorators can be configured directly in ehcache.xml. The decorators will be created and added to the CacheManager.

It accepts the name of a concrete class that extends net.sf.ehcache.constructs.CacheDecoratorFactory

The properties will be parsed according to the delimiter (default is comma ',') and passed to the concrete factory's
`createDecoratedEhcache(Ehcache cache, Properties properties)` method along with the reference to the owning cache.

It is configured as per the following example:

~~~ xml
<cacheDecoratorFactory
    class="com.company.SomethingCacheDecoratorFactory"
    properties="property1=36 ..." />
~~~

Note that from version 2.2, decorators can be configured against the `defaultCache`. This is very useful for frameworks
 like Hibernate that add caches based on the `defaultCache`.

## Adding decorated caches to the CacheManager

Having created a decorator programmatically it is generally useful to put it in a place where multiple threads may access it.
Note that decorators created via configuration in ehcache.xml have already been added to the `CacheManager`.

### Using `CacheManager.replaceCacheWithDecoratedCache()`

A built-in way is to replace the Cache in CacheManager with the decorated one. This is achieved as in the following
example:

~~~ java
cacheManager.replaceCacheWithDecoratedCache(cache, newBlockingCache);
~~~

The `CacheManager` `{replaceCacheWithDecoratedCache}` method requires that the decorated cache be built from
 the underlying cache from the same name.

Note that any overwridden Ehcache methods will take on new behaviours without casting, as per the normal rules
 of Java. Casting is only required for new methods that the decorator introduces.

Any calls to get the cache out of the CacheManager now return the decorated one.

A word of caution. This method should be called in an appropriately synchronized init style method before multiple threads
attempt to use it. All threads must be referencing the same decorated cache. An example of a suitable init method is
found in `CachingFilter`:

~~~ java
/**
 * The cache holding the web pages. Ensure that all threads for a given cache name
 * are using the same instance of this.
 */
private BlockingCache blockingCache;
/**
 * Initialises blockingCache to use
 *
 * @throws CacheException The most likely cause is that a cache has not been
 *                        configured in Ehcache's configuration file ehcache.xml
 *                        for the filter name
 */
public void doInit() throws CacheException {
  synchronized (this.getClass()) {
    if (blockingCache == null) {
      final String cacheName = getCacheName();
      Ehcache cache = getCacheManager().getEhcache(cacheName);
      if (!(cache instanceof BlockingCache)) {
        //decorate and substitute
        BlockingCache newBlockingCache = new BlockingCache(cache);
        getCacheManager().replaceCacheWithDecoratedCache(cache, newBlockingCache);
      }
      blockingCache = (BlockingCache) getCacheManager().getEhcache(getCacheName());
    }
  }
}
~~~

~~~ java
Ehcache blockingCache = singletonManager.getEhcache("sampleCache1");
~~~

The returned cache will exhibit the decorations.

### Using `CacheManager.addDecoratedCache()`

Sometimes you want to add a decorated cache but retain access to the underlying cache.

The way to do this is to create a decorated cache and then call `cache.setName(new_name)` and then add it to `CacheManager`
with `CacheManager.addDecoratedCache()`.

~~~ java
/**
 * Adds a decorated {@link Ehcache} to the CacheManager. This method neither creates
 * the memory/disk store nor initializes the cache. It only adds the cache reference
 * to the map of caches held by this cacheManager.
 * <p/>
 * It is generally required that a decorated cache, once constructed, is made available
 * to other execution threads. The simplest way of doing this is to either add it to
 * the cacheManager with a different name or substitute the original cache with the
 * decorated one.
 * <p/>
 * This method adds the decorated cache assuming it has a different name. If another
 * cache (decorated or not) with the same name already exists, it will throw
 * {@link ObjectExistsException}. For replacing existing
 * cache with another decorated cache having same name, please use
 * {@link #replaceCacheWithDecoratedCache(Ehcache, Ehcache)"/>
 * <p/>
 * Note that any overridden Ehcache methods by the decorator will take on new
 * behaviours without casting. Casting is only required for new methods that the
 * decorator introduces. For more information see the well known Gang of Four
 * Decorator pattern.
 *
 * @param decoratedCache
 * @throws ObjectExistsException
 *             if another cache with the same name already exists.
 */
public void addDecoratedCache(Ehcache decoratedCache) throws ObjectExistsException {
~~~

## Built-in Decorators

### BlockingCache <a name="BlockingCache"/>

A blocking decorator for an Ehcache, backed by a {@link Ehcache}.

It allows concurrent read access to elements already in the cache. If the element is null, other
 reads will block until an element with the same key is put into the cache.
 This is useful for constructing read-through or self-populating caches.
 BlockingCache is used by `CachingFilter`.

### SelfPopulatingCache <a name="SelfPopulatingCache"/>

A selfpopulating decorator for Ehcache that creates entries on demand.

Clients of the cache simply call it without needing knowledge of whether
 the entry exists in the cache. If null the entry is created.
 The cache is designed to be refreshed. Refreshes operate on the backing cache, and do not
 degrade performance of get calls.

SelfPopulatingCache extends BlockingCache. Multiple threads attempting to access a null element will block
 until the first thread completes. If refresh is being called the threads do not block - they return the stale data.
 This is very useful for engineering highly scalable systems.

### Caches with Exception Handling

These are decorated. See [Cache Exception Handlers](/documentation/2.7/apis/cache-exception-handlers) for full details. 
