### 1. **Race Condition**

A **race condition** occurs when two or more threads or tasks access shared data simultaneously, and the result of the execution depends on the timing of the threads. If the execution order is not synchronized, the shared data may end up in an inconsistent or unpredictable state.

#### Example of a Race Condition

Imagine two threads trying to increment a shared counter. Without synchronization, both threads might read the same value of the counter, increment it, and write it back, resulting in the counter being incremented only once instead of twice.

```csharp
public class RaceConditionExample
{
    private int _counter = 0;

    public void Increment()
    {
        _counter++;
    }

    public void SimulateRaceCondition()
    {
        Thread t1 = new Thread(Increment);
        Thread t2 = new Thread(Increment);

        t1.Start();
        t2.Start();

        t1.Join();
        t2.Join();

        Console.WriteLine($"Final Counter Value: {_counter}");
    }
}
```

In the above code:
- The `Increment()` method increments the `_counter`.
- Two threads (`t1` and `t2`) are started, both calling the `Increment()` method.
- Since there is no synchronization, both threads may read the same value of `_counter`, increment it, and write the same result back.

**Possible outcome**: Even though two threads increment the counter, the final value might still be 1 instead of 2 because of the race condition.

#### Fixing the Race Condition

To prevent race conditions, synchronization is required. We can use the `lock` keyword to ensure that only one thread can access the critical section at a time:

```csharp
public class RaceConditionFixed
{
    private int _counter = 0;
    private readonly object _lock = new object();

    public void Increment()
    {
        lock (_lock)
        {
            _counter++;
        }
    }

    public void SimulateFixedRaceCondition()
    {
        Thread t1 = new Thread(Increment);
        Thread t2 = new Thread(Increment);

        t1.Start();
        t2.Start();

        t1.Join();
        t2.Join();

        Console.WriteLine($"Final Counter Value: {_counter}");
    }
}
```

Now the counter will be incremented correctly since only one thread can enter the `lock` block at a time, preventing the race condition.

---

### 2. **Deadlock**

A **deadlock** occurs when two or more threads are waiting for each other to release a resource, and none of them can proceed because they are holding locks that the others need.

#### Example of a Deadlock

Let’s create a deadlock scenario with two threads and two locks. Each thread tries to acquire both locks but in a different order.

```csharp
public class DeadlockExample
{
    private readonly object _lock1 = new object();
    private readonly object _lock2 = new object();

    public void Thread1()
    {
        lock (_lock1)
        {
            Console.WriteLine("Thread 1 acquired Lock 1");
            Thread.Sleep(100);  // Simulate some work
            lock (_lock2)
            {
                Console.WriteLine("Thread 1 acquired Lock 2");
            }
        }
    }

    public void Thread2()
    {
        lock (_lock2)
        {
            Console.WriteLine("Thread 2 acquired Lock 2");
            Thread.Sleep(100);  // Simulate some work
            lock (_lock1)
            {
                Console.WriteLine("Thread 2 acquired Lock 1");
            }
        }
    }

    public void SimulateDeadlock()
    {
        Thread t1 = new Thread(Thread1);
        Thread t2 = new Thread(Thread2);

        t1.Start();
        t2.Start();

        t1.Join();
        t2.Join();
    }
}
```

In this example:
- Thread 1 acquires `_lock1` and waits for `_lock2`.
- Thread 2 acquires `_lock2` and waits for `_lock1`.

**Deadlock scenario**: Both threads are now waiting for each other to release their respective locks, causing a deadlock where neither can proceed.

#### Fixing the Deadlock

One way to fix this deadlock is to ensure that both threads acquire locks in the same order. If both threads always acquire `_lock1` first and then `_lock2`, they will not block each other.

```csharp
public class DeadlockFixed
{
    private readonly object _lock1 = new object();
    private readonly object _lock2 = new object();

    public void Thread1()
    {
        lock (_lock1)
        {
            Console.WriteLine("Thread 1 acquired Lock 1");
            Thread.Sleep(100);  // Simulate some work
            lock (_lock2)
            {
                Console.WriteLine("Thread 1 acquired Lock 2");
            }
        }
    }

    public void Thread2()
    {
        lock (_lock1)  // Changed order to match Thread1
        {
            Console.WriteLine("Thread 2 acquired Lock 1");
            Thread.Sleep(100);  // Simulate some work
            lock (_lock2)
            {
                Console.WriteLine("Thread 2 acquired Lock 2");
            }
        }
    }

    public void SimulateFixedDeadlock()
    {
        Thread t1 = new Thread(Thread1);
        Thread t2 = new Thread(Thread2);

        t1.Start();
        t2.Start();

        t1.Join();
        t2.Join();
    }
}
```

In the fixed version, both `Thread1` and `Thread2` acquire `_lock1` before `_lock2`, preventing the deadlock.

---

### Conclusion

- **Race Condition**: Multiple threads or tasks access shared resources in an unsynchronized manner, causing incorrect results. It can be fixed using synchronization mechanisms like `lock`.
- **Deadlock**: Two or more threads are waiting on each other to release locks they need, causing a permanent blocking state. It can be fixed by ensuring consistent lock acquisition order or by using mechanisms like `Monitor.TryEnter` with timeouts.