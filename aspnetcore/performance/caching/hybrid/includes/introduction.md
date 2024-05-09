The [`HybridCache`](https://source.dot.net/#Microsoft.Extensions.Caching.Hybrid/Runtime/HybridCache.cs,8c0fe94693d1ac8d) API bridges some gaps in the <xref:Microsoft.Extensions.Caching.Distributed.IDistributedCache> and <xref:Microsoft.Extensions.Caching.Memory.IMemoryCache> APIs. `HybridCache` has the following features that these APIs don't have:

* A unified API for both in-process and out-of-process caching.

  `HybridCache` is designed to be a drop-in replacement for existing `IDistributedCache` and `IMemoryCache` usage, and it provides a 
simple API for adding new caching code.

* Stampede and thundering herd protection.

  *Cache stampede* happens when a frequently used cache entry is revoked, and too many requests try to repopulate the same cache entry at the same time. *Thundering herd* is similar: a burst of requests for the same response that isn't already in a cache entry. `HybridCache` combines concurrent operations, ensuring that all requests for a given response wait for the first request to populate the cache.

* Configurable serialization.

  Serialization is configured as part of registering the service, with support for type-specific and generalized serializers via the `WithSerializer` and `WithSerializerFactory` methods, chained from the `AddHybridCache` call. By default, the library handles `string` and `byte[]` internally, and uses `System.Text.Json` for everything else. And it can be configured for other types of serializers, such as protobuf or XML.



  

