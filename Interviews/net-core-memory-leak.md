Memory leaks in .NET Core occur when an application unintentionally retains objects in memory that are no longer needed, preventing the garbage collector (GC) from reclaiming that memory. Over time, this can lead to increased memory usage, degraded performance, and, in extreme cases, application crashes. Below, we'll explore common causes of memory leaks in .NET Core applications and provide solutions to mitigate them.

### 1. **Common Causes of Memory Leaks in .NET Core**
#### a. **Unmanaged Resources Not Released**
- **Cause:** Unmanaged resources, such as file handles, database connections, and network sockets, are not automatically cleaned up by the GC.
- **Example:** Using `FileStream`, `Socket`, or `SqlConnection` objects without closing or disposing of them properly.
  
**Solution:**
- Implement the `IDisposable` interface and use `using` statements to ensure resources are cleaned up when no longer needed.

```csharp
using (var stream = new FileStream("file.txt", FileMode.Open))
{
    // Use the stream
}  // Automatically disposes the stream
```

- For classes holding unmanaged resources, explicitly implement `IDisposable` and call `Dispose()` to release the resources.

```csharp
public class ResourceHolder : IDisposable
{
    private IntPtr unmanagedResource;
    
    public void Dispose()
    {
        // Release unmanaged resource
        if (unmanagedResource != IntPtr.Zero)
        {
            FreeUnmanagedResource(unmanagedResource);
            unmanagedResource = IntPtr.Zero;
        }
        GC.SuppressFinalize(this);  // Prevent the finalizer from running
    }

    ~ResourceHolder()
    {
        Dispose();
    }
}
```

#### b. **Static Events and Event Handlers**
- **Cause:** When objects subscribe to events and the publisher is long-lived (e.g., static events), the subscriber remains in memory as long as the event publisher is alive. This prevents the GC from collecting the subscriber, leading to memory leaks.
  
**Solution:**
- **Unsubscribe from events** when the subscriber no longer needs to listen for events.
  
```csharp
public class EventConsumer
{
    public EventConsumer()
    {
        EventPublisher.SomeEvent += HandleEvent;
    }

    private void HandleEvent(object sender, EventArgs e)
    {
        // Handle event
    }

    public void Unsubscribe()
    {
        EventPublisher.SomeEvent -= HandleEvent;
    }
}
```

- Use **weak event patterns** to avoid holding strong references to event subscribers.

#### c. **Static Collections or Caches**
- **Cause:** Statically referenced collections (e.g., dictionaries, lists) that accumulate objects without limit can cause a memory leak, as objects in static collections persist for the lifetime of the application.
  
**Solution:**
- Use **memory-sensitive caching strategies**, such as `MemoryCache` with expiration policies, to limit memory usage.
  
```csharp
var cache = new MemoryCache(new MemoryCacheOptions());

cache.Set("key", value, TimeSpan.FromMinutes(5));  // Expire after 5 minutes
```

- Regularly clean up static collections by removing unused or stale items.

#### d. **Long-lived Objects Referencing Short-lived Objects**
- **Cause:** If a long-lived object (e.g., a singleton service) holds references to short-lived objects (e.g., per-request services), those short-lived objects won't be garbage collected until the long-lived object is collected, resulting in a memory leak.

**Solution:**
- **Avoid long-lived references** to short-lived objects. Use dependency injection and ensure that transient or scoped services are not stored in long-lived objects like singletons.
  
```csharp
public class LongLivedService
{
    private IServiceScopeFactory _scopeFactory;

    public LongLivedService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public void Process()
    {
        using (var scope = _scopeFactory.CreateScope())
        {
            var transientService = scope.ServiceProvider.GetRequiredService<ITransientService>();
            transientService.DoWork();
        }
    }
}
```

#### e. **WeakReferences Not Used When Necessary**
- **Cause:** Holding strong references to objects that are infrequently used, but not released when no longer needed, can lead to memory leaks.
  
**Solution:**
- Use **weak references** to allow the GC to collect objects when they are no longer in use, even if there are still references to them.
  
```csharp
WeakReference<MyClass> weakReference = new WeakReference<MyClass>(myObject);
```

### 2. **Diagnosing Memory Leaks**
Memory leaks can be difficult to identify, so .NET Core provides tools for memory diagnostics.

#### a. **dotnet-counters**
`dotnet-counters` can track key performance counters, such as memory usage, GC activity, and object allocations, to help monitor the health of the application in real time.

```bash
dotnet-counters monitor System.Runtime
```

#### b. **dotnet-gcdump**
This tool captures a snapshot of the garbage collector heap, which can be analyzed to determine which objects are consuming memory and why they haven’t been collected.

```bash
dotnet-gcdump collect -p <process-id>
```

You can analyze the `.gcdump` file using Visual Studio or PerfView to see the object graph and understand retention issues.

#### c. **dotnet-trace**
`dotnet-trace` can help diagnose memory leaks by recording detailed trace events, including allocations and GC operations.

```bash
dotnet-trace collect -p <process-id>
```

#### d. **Visual Studio Diagnostic Tools**
The **Memory Profiler** in Visual Studio allows you to take snapshots of heap usage, track object references, and compare snapshots to identify memory leaks.
  
1. Open **Diagnostic Tools** in Visual Studio.
2. Start a **memory usage session**.
3. Take **snapshots** periodically during execution.
4. Analyze differences between snapshots to detect objects that are not being collected by the GC.

#### e. **GC Logs**
Enable **GC logging** for detailed information about collections and memory pressure. This can help detect if certain objects are being collected or if memory is continuously growing.

```json
{
  "runtimeOptions": {
    "configProperties": {
      "System.GC.Server": true,
      "System.GC.Concurrent": true,
      "System.GC.Log": true
    }
  }
}
```

### 3. **Best Practices for Preventing Memory Leaks**
#### a. **Use `IDisposable` and `using` Statements**
Always ensure unmanaged resources are properly disposed by implementing `IDisposable`. Use `using` statements to automatically clean up resources when they are no longer needed.

#### b. **Avoid Capturing Variables in Long-Lived Closures**
Capturing variables in closures can unintentionally keep them alive longer than necessary. Avoid capturing short-lived variables in long-lived delegates, lambda expressions, or async operations.

```csharp
Action action = () => Console.WriteLine(shortLivedObject.ToString());  // Potential leak
```

#### c. **Use Weak References for Caching**
In scenarios where caching is required, but objects should be eligible for GC, use weak references or a cache library with built-in eviction policies (e.g., `MemoryCache`).

#### d. **Avoid Long-Lived Object References**
Ensure that long-lived objects (e.g., singletons) do not hold onto references to short-lived objects unless necessary. Transient or scoped services should not be referenced in singleton services.

#### e. **Unsubscribe from Events**
Always unsubscribe from events when an object is no longer needed, especially for long-lived or static event publishers. Implement event handler cleanup in `Dispose()`.

#### f. **Minimize Large Object Heap Fragmentation**
Large objects are allocated in the LOH, which is not compacted frequently, leading to fragmentation. Avoid allocating large arrays or buffers repeatedly, and use **object pooling** techniques where appropriate.

#### g. **Use Object Pools**
When creating large numbers of objects that are expensive to allocate, consider using **object pooling** to reuse objects and avoid memory fragmentation or excessive GC pressure.

### Conclusion
Memory leaks in .NET Core can be subtle and difficult to detect, but they can have serious performance impacts if not addressed. Following best practices like disposing of unmanaged resources, using weak references for caches, and regularly profiling the application can prevent memory leaks. Tools like `dotnet-gcdump`, `dotnet-trace`, and Visual Studio's memory profiler are invaluable for diagnosing and resolving these issues.