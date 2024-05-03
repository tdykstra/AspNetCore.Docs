---
ms.topic: include
author: mgravell
ms.author: marcgravell
ms.date: 05/02/2024
---
### New `HybridCache` library

The [`HybridCache`](https://source.dot.net/#Microsoft.Extensions.Caching.Hybrid/Runtime/HybridCache.cs,8c0fe94693d1ac8d) API bridges some gaps in the existing <xref:Microsoft.Extensions.Caching.Distributed.IDistributedCache> and <xref:Microsoft.Extensions.Caching.Memory.IMemoryCache> APIs. It also adds new capabilities, such as:

* **"Stampede" protection** to prevent parallel fetches of the same work.
* Configurable serialization.

`HybridCache` is designed to be a drop-in replacement for existing `IDistributedCache` and `IMemoryCache` usage, and it provides a simple API for adding new caching code. It provides a unified API for both in-process and out-of-process caching.

To understand the `HybridCache` API, compare it to `IDistributedCache` code. Here's an example of a service that uses `IDistributedCache`:

```cs
public class SomeService(IDistributedCache cache)
{
    public async Task<SomeInformation> GetSomeInformationAsync
        (string name, int id, CancellationToken token = default)
    {
        var key = $"someinfo:{name}:{id}"; // Unique key for this combination.
        var bytes = await cache.GetAsync(key, token); // Try to get from cache.
        SomeInformation info;
        if (bytes is null)
        {
            // Cache miss; get the data from the real source.
            info = await SomeExpensiveOperationAsync(name, id, token);

            // Serialize and cache it.
            bytes = SomeSerializer.Serialize(info);
            await cache.SetAsync(key, bytes, token);
        }
        else
        {
            // Cache hit; deserialize it.
            info = SomeSerializer.Deserialize<SomeInformation>(bytes);
        }
        return info;
    }

    // This is the work we're trying to cache.
    private async Task<SomeInformation> SomeExpensiveOperationAsync(string name, int id,
        CancellationToken token = default)
    { /* ... */ }
}
```

That's a lot of work to get right each time, including things like serialization. And in the "cache miss" scenario, you could end up with multiple concurrent threads, *all* getting a cache miss, *all* fetching the underlying data, *all* serializing it, and *all* sending that data to the cache.

To simplify and improve this code with `HybridCache`, we first need to add the new library `Microsoft.Extensions.Caching.Hybrid`:

``` xml
<PackageReference Include="Microsoft.Extensions.Caching.Hybrid" Version="9.0.0" />
```

Register the `HybridCache` service, like you would register an `IDistributedCache` implementation:

```cs
services.AddHybridCache(); // Not shown: optional configuration API.
```

Now most caching concerns can be offloaded to `HybridCache`:

```cs
public class SomeService(HybridCache cache)
{
    public async Task<SomeInformation> GetSomeInformationAsync
        (string name, int id, CancellationToken token = default)
    {
        return await cache.GetOrCreateAsync(
            $"someinfo:{name}:{id}", // Unique key for this combination.
            async cancel => await SomeExpensiveOperationAsync(name, id, cancel),
            token: token
        );
    }
}
```

The `HybridCache` implementation deals with everything related to caching, including concurrent operation handling. The `cancel` token here represents the combined cancellation of *all* concurrent callers&mdash;not just the cancellation of the caller we can see (`token`). High throughput scenarios, can be further
optimized by using the `TState` pattern, to avoid some overhead from "captured" variables and per-instance callbacks:

```cs
public class SomeService(HybridCache cache)
{
    public async Task<SomeInformation> GetSomeInformationAsync(string name, int id, CancellationToken token = default)
    {
        return await cache.GetOrCreateAsync(
            $"someinfo:{name}:{id}", // unique key for this combination
            (name, id), // all of the state we need for the final call, if needed
            static async (state, token) =>
                await SomeExpensiveOperationAsync(state.name, state.id, token),
            token: token
        );
    }
}
```

`HybridCache` uses the configured `IDistributedCache` implementation, if any, for secondary out-of-process caching, for example, using
Redis. But even without an `IDistributedCache`, the `HybridCache` service will still provide in-process caching and "stampede" protection. For more information, see     [Distributed caching](https://learn.microsoft.com/aspnet/core/performance/caching/distributed).

### A note on object reuse

Because a lot of `HybridCache` usage will be adapted from existing `IDistributedCache` code, we need to be mindful that *existing* code will usually
be deserializing every call - which means that concurrent callers will get separate object instances that cannot interact and are inherently
thread-safe. To avoid introducing concurrency bugs into code, `HybridCache` preserves this behaviour by default, but if your scenario is itself
thread-safe (either because the types are fundamentally immutable, or because you're just *not mutating them*), you can hint to `HybridCache`
that it can safely reuse instances by marking the type (`SomeInformation` in this case) as `sealed` and using the `[ImmutableObject(true)]` annotation,
which can significantly reduce per-call deserialization overheads of CPU and object allocations.

### Other `HybridCache` features

As you might expect for parity with `IDistributedCache`, `HybridCache` supports explicit removal by key (`cache.RemoveKeyAsync(...)`). `HybridCache`
also introduces new optional APIs for `IDistributedCache` implementations, to avoid `byte[]` allocations (this feature is implemented
by the preview versions of `Microsoft.Extensions.Caching.StackExchangeRedis` and `Microsoft.Extensions.Caching.SqlServer`).

Serialization is configured as part of registering the service, with support for type-specific and generalized serializers via the
`.WithSerializer(...)` and `.WithSerializerFactory(...)` methods, chained from the `AddHybridCache(...)` call. By default, the library
handles `string` and `byte[]` internally, and uses `System.Text.Json` for everything else, but if you want to use protobuf, xml, or anything
else: that's easy to do.

`HybridCache` includes support for older .NET runtimes, down to .NET Framework 4.7.2 and .NET Standard 2.0.

Outstanding `HybridCache` work includes:

- support for "tagging" (similar to how tagging works for "Output Cache"), allowing invalidation of entire *categories* of data
- backend-assisted cache invalidation, for backends that can provide suitable change notifications
- relocation of the core abstractions to `Microsoft.Extensions.Caching.Abstractions`