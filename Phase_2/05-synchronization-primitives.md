---
phase: 2
topic: 5
title: Synchronization Primitives
tags: [csharp, threading, synchronization, lock, monitor, mutex, semaphoreslim, readerwriterlockslim, manualreseteventslim]
date: 2026-06-03
status: in-progress
---

# Synchronization Primitives #threading #synchronization

## Table of Contents
- [[#Overview]]
- [[#lock]]
- [[#Monitor]]
- [[#Mutex]]
- [[#SemaphoreSlim]]
- [[#SafeQueue Example]]
- [[#Common Pitfalls]]
- [[#Interview Questions]]
- [[#ReaderWriterLockSlim]]
- [[#ManualResetEventSlim]]

---

## Overview

Synchronization primitives are used to coordinate access to shared resources across multiple threads. Each primitive has different trade-offs between simplicity, flexibility, and async compatibility.

| Primitive | Use case | Async-friendly |
|---|---|---|
| `lock` / `Monitor` | Mutual exclusion, Wait/Pulse signaling | ❌ |
| `SemaphoreSlim` | Bounded concurrency, async gates | ✅ |
| `Interlocked` | Lock-free atomic operations | ✅ |
| `ManualResetEventSlim` | One-time or resettable signal | ✅ |
| `ReaderWriterLockSlim` | Multiple readers, single writer | ❌ |
| `Mutex` | Cross-process locking | ❌ |
| `SpinLock` / `SpinWait` | Very short critical sections | ❌ |

---

## lock

### What it compiles to

`lock` is syntactic sugar. This:

```csharp
lock (_lock)
{
    // work
}
```

compiles to:

```csharp
bool lockTaken = false;
try
{
    Monitor.Enter(_lock, ref lockTaken);
    // work
}
finally
{
    if (lockTaken) Monitor.Exit(_lock);
}
```

> [!NOTE]
> The `lockTaken` guard exists because a `ThreadAbortException` can fire in the tiny window between `Monitor.Enter` acquiring the lock and execution reaching the `try` block. Without `lockTaken`, `finally` wouldn't know whether to call `Exit`, potentially leaking the lock forever.

### What NOT to lock on

| Target | Why it's wrong |
|---|---|
| `this` | The object reference is visible externally — other code can lock on the same instance, causing unexpected deadlocks |
| `string` | String literals are interned — identical strings across the AppDomain share the same reference, so unrelated code can deadlock you |
| `typeof(MyClass)` | `Type` objects are shared across the AppDomain, same problem as string interning |

Always lock on a **private, dedicated `object` field**:

```csharp
private readonly object _lock = new object();
```

### Cannot use await inside lock

```csharp
lock (_lock)
{
    await SomethingAsync(); // compiler error
}
```

`await` may resume on a different thread, but `Monitor` requires the **same thread** to release the lock it acquired. These are incompatible.

Use `SemaphoreSlim(1, 1)` with `WaitAsync()` instead for async mutual exclusion.

---

## Monitor

`Monitor` is what `lock` compiles to, but exposes additional capabilities: `Wait`, `Pulse`, `PulseAll`, and `TryEnter`.

### Monitor.Enter vs lock

`Monitor.Enter` behaves identically to `lock` but is manual — you must write the `try/finally` yourself. The risk is forgetting `Exit` on exception paths. Prefer `lock` unless you need `TryEnter`.

### Monitor.TryEnter

```csharp
bool lockTaken = false;
Monitor.TryEnter(_lock, TimeSpan.FromSeconds(1), ref lockTaken);
if (lockTaken)
{
    try { /* work */ }
    finally { Monitor.Exit(_lock); }
}
else
{
    // lock not acquired — do something else
}
```

> [!NOTE]
> `ref bool lockTaken` and `TryEnter` with timeout solve different problems:
> - `lockTaken` — safety against async exceptions between `Enter` and `try`
> - `TryEnter(timeout)` — non-blocking, do other work if lock isn't free in time

### Monitor.Wait and Monitor.Pulse

Used for **thread signaling** — a thread waits for a condition to become true, another thread signals when it does.

**Two internal queues:**

| Queue | Contains |
|---|---|
| Wait queue | Threads that called `Monitor.Wait` — suspended, waiting for `Pulse` |
| Ready queue | Threads that were pulsed and are now competing for the lock |

**`Monitor.Wait(_lock)`** — must be called while holding the lock:
1. Atomically releases the lock
2. Suspends the thread (moved to wait queue)
3. When pulsed, moves to ready queue
4. Re-acquires the lock before returning

**`Monitor.Pulse(_lock)`** — wakes one thread from the wait queue → ready queue.

**`Monitor.PulseAll(_lock)`** — wakes all threads from the wait queue.

> [!WARNING]
> `Wait` and `Pulse`/`PulseAll` must be called from inside a `lock` block. Calling them without holding the monitor throws `SynchronizationLockException`.

### Always use while, never if

```csharp
// WRONG
if (_queue.Count == 0)
    Monitor.Wait(_lock);

// CORRECT
while (_queue.Count == 0)
    Monitor.Wait(_lock);
```

With multiple waiters, `PulseAll` wakes all threads but only one can acquire the lock. By the time the second thread acquires the lock, the first may have already consumed the item. The `while` re-checks the condition, the `if` does not.

This is called a **spurious wakeup guard**.

### Pulse vs PulseAll

| | `Pulse` | `PulseAll` |
|---|---|---|
| Wakes | 1 thread | All waiting threads |
| Risk | Wrong thread woken, no progress | Thundering herd, wasted wakeups |
| Safe with | Single producer + single consumer | Multiple producers and/or consumers |

> [!NOTE]
> `Monitor` has a **single wait queue** — it cannot selectively wake producers or consumers. This is a key limitation. `SemaphoreSlim` solves this with separate signals.

### What happens to the thread during Wait

| Thread type | Behavior |
|---|---|
| `new Thread(...)` | Suspended by OS — alive but parked, no CPU used |
| ThreadPool thread | Suspended — pool may spin up replacement thread. **Avoid blocking ThreadPool threads.** |
| `async/await` | Use `SemaphoreSlim.WaitAsync()` instead — releases thread back to pool |

---

## Mutex

`Mutex` is like `lock`/`Monitor` but operates at the **OS level**, enabling synchronization across multiple processes or app instances.

### lock vs Mutex

| | `lock` / `Monitor` | `Mutex` |
|---|---|---|
| Scope | Single process | Cross-process |
| Lives in | CLR (user-space) | OS kernel |
| Speed | Fast | Slow (kernel transition) |
| Named | ❌ | ✅ |
| Async-friendly | ❌ | ❌ |

### Creating vs Acquiring

These are two separate steps — easy to confuse:

```csharp
// Creating — gets a handle to the OS object (does NOT acquire it)
var mutex = new Mutex(false, "MyAppMutex");

// Acquiring — blocks until the mutex is available
mutex.WaitOne();

// Releasing
mutex.ReleaseMutex();
```

> [!NOTE]
> The first `bool` parameter of `new Mutex(initiallyOwned, name)` controls whether the **creating thread immediately owns** the mutex. Pass `false` unless you explicitly want to own it on creation.

### Named Mutex — single instance app pattern

A named `Mutex` is a system-wide OS object identified by its name. Any process that uses the same name refers to the **same OS object**.

```csharp
var mutex = new Mutex(false, "MyAppSingleInstance");
bool acquired = false;

try
{
    try
    {
        acquired = mutex.WaitOne(0); // non-blocking — returns immediately
    }
    catch (AbandonedMutexException)
    {
        acquired = true; // we still own it — previous instance crashed
        Console.WriteLine("Warning: previous instance crashed, state may be corrupted.");
    }

    if (!acquired)
    {
        Console.WriteLine("Another instance is already running. Exiting.");
        return;
    }

    Console.WriteLine("Application started. Running... (press Enter to exit)");
    Console.ReadLine();
}
finally
{
    if (acquired)
        mutex.ReleaseMutex();

    mutex.Dispose();
}
```

> [!NOTE]
> Use `WaitOne(0)` for a **non-blocking attempt** — returns `true` if acquired, `false` if already taken. Without the timeout overload, `WaitOne()` blocks indefinitely.

### AbandonedMutexException

If a thread owns a `Mutex` and **terminates without releasing it**, the Mutex becomes *abandoned*.

- The next thread calling `WaitOne` **does acquire** the Mutex
- But receives `AbandonedMutexException` as a warning
- Signals that shared state protected by this Mutex may be **corrupted**
- You can catch it, log a warning, and decide whether to proceed

> [!WARNING]
> Do NOT use `using` on `Mutex` for release — `Dispose()` and `ReleaseMutex()` are separate concerns. Always release with `ReleaseMutex()` in a `finally` block, then `Dispose()`.

### Why Mutex is slow

`lock`/`Monitor` operate entirely in **user-space** (CLR manages them). `Mutex` is a kernel object — every acquire/release requires a **user-mode → kernel-mode transition**, which involves a context switch managed by the OS. This makes it orders of magnitude slower than `lock`.

Use `Mutex` only when cross-process synchronization is actually needed. For single-process use, prefer `lock`.

---

## SafeQueue Example

Bounded producer/consumer queue using `Monitor.Wait` and `PulseAll`:

```csharp
class SafeQueue<T>
{
    private readonly Queue<T> _queue = new Queue<T>();
    private readonly object _lock = new object();
    private readonly int _maxCapacity = 5;

    public void Enqueue(T item)
    {
        lock (_lock)
        {
            while (_queue.Count >= _maxCapacity)
                Monitor.Wait(_lock);

            _queue.Enqueue(item);
            Monitor.PulseAll(_lock); // wake all — consumers and producers re-check their condition
        }
    }

    public T Dequeue()
    {
        lock (_lock)
        {
            while (_queue.Count == 0)
                Monitor.Wait(_lock);

            var item = _queue.Dequeue();
            Monitor.PulseAll(_lock);
            return item;
        }
    }
}
```

> [!NOTE]
> Use `PulseAll` when multiple producers or consumers are possible. With a single producer + single consumer, `Pulse` is sufficient and avoids the thundering herd.

> [!WARNING]
> Setting `_maxCapacity = 0` causes an instant deadlock — producer immediately waits (queue always full), consumer immediately waits (queue always empty), nobody ever calls Pulse.

---

## Common Pitfalls

| Pitfall | Description |
|---|---|
| `if` instead of `while` on `Wait` | Condition may no longer be true when thread wakes — always use `while` |
| `await` inside `lock` | Compiler error — use `SemaphoreSlim` instead |
| Locking on `this` / `string` / `Type` | Shared references cause unexpected deadlocks |
| Forgetting `Monitor.Exit` | Use `lock` keyword or always wrap in `try/finally` |
| Unbounded `Wait` with no timeout | Thread hangs forever if `Pulse` is never called |
| `_maxCapacity = 0` | Instant deadlock — both producer and consumer wait immediately |
| Single `Pulse` with multiple consumers | Wrong thread may be woken, correct thread stays sleeping |

### Deadlock example

```csharp
// Thread A: locks _lock1 then tries _lock2
public void MethodA()
{
    lock (_lock1) { lock (_lock2) { } }
}

// Thread B: locks _lock2 then tries _lock1
public void MethodB()
{
    lock (_lock2) { lock (_lock1) { } }
}
```

Thread A holds `_lock1`, waiting for `_lock2`. Thread B holds `_lock2`, waiting for `_lock1`. Neither can proceed.

**Mitigation:** always acquire multiple locks in a consistent global order across all threads.

### Lock convoy

Lock convoy occurs when multiple threads contend for the same lock in a tight loop. Waiting threads are repeatedly woken up by the OS, fail to acquire the lock (already taken by an already-running thread), and go back to sleep — wasting context switches without doing useful work.

**Mitigations:** `ConcurrentQueue<T>` / `ConcurrentDictionary<T>`, `ReaderWriterLockSlim` for read-heavy workloads, reducing lock scope.

---

## Interview Questions

1. What does `lock` compile to?
2. Why can't you use `await` inside a `lock` block?
3. Why must you never lock on `this`, a `string`, or a `Type`?
4. What happens to the lock if an exception is thrown inside a `lock` block?
5. What is the `ref bool lockTaken` pattern and why does it exist?
6. Walk through exactly what `Monitor.Wait` does to the thread and the lock.
7. `Monitor.Pulse` vs `Monitor.PulseAll` — when to use each?
8. Why must `Monitor.Wait` always be inside a `while` loop?
9. Describe a minimal two-thread deadlock using `lock`. How do you prevent it?
10. What is lock convoy and how do you mitigate it?
11. What is the difference between `Mutex` and `lock`? When would you choose `Mutex`?
12. What does `AbandonedMutexException` mean, and does the throwing thread own the Mutex?
13. Why is `Mutex` slower than `lock`?
14. What is a named `Mutex` and where does it live?

---

See also: [[02-async-await-internals]] | [[06-deadlocks-race-conditions]]

---

## SemaphoreSlim

A lightweight synchronization primitive that limits the number of threads (or async tasks) that can access a resource concurrently. Unlike `lock` and `Mutex`, it supports `async/await` and allows more than 1 concurrent caller.

### Constructor

```csharp
new SemaphoreSlim(initialCount)           // same as (initialCount, initialCount)
new SemaphoreSlim(initialCount, maxCount)
```

| Parameter | Meaning |
|---|---|
| `initialCount` | Available permits at creation time |
| `maxCount` | Upper bound — `Release()` throws `SemaphoreFullException` if exceeded |

```csharp
new SemaphoreSlim(3)     // 3 permits, max 3 — throttle pattern
new SemaphoreSlim(0, 1)  // starts locked — signaling pattern
```

### Key Methods

| Method | Description |
|---|---|
| `Wait()` | Blocks the thread until a permit is available |
| `WaitAsync()` | Suspends the async method, thread returns to pool |
| `Release()` | Increments the counter by 1 |
| `Release(n)` | Increments the counter by n |
| `CurrentCount` | How many permits are currently available |

> [!WARNING]
> Always call `Release()` in a `finally` block — if the protected code throws, the permit must still be returned

### Throttle Pattern

Limit concurrent access to a resource (e.g. external API, DB connection pool).

```csharp
private readonly SemaphoreSlim _semaphore = new SemaphoreSlim(5, 5);

public async Task CallApiAsync()
{
    await _semaphore.WaitAsync();
    try
    {
        await httpClient.GetAsync("...");
    }
    finally
    {
        _semaphore.Release();
    }
}
```

Flow: `WaitAsync()` → do work → `Release()` (same logical owner)

### Signaling Pattern

One task waits, another signals when ready. Semaphore starts at `initialCount = 0`.

```csharp
var semaphore = new SemaphoreSlim(0, 1);

var consumer = Task.Run(async () =>
{
    await semaphore.WaitAsync(); // blocks until signaled
    Console.WriteLine("got the signal");
    // no Release() needed — this is a one-shot signal
});

var producer = Task.Run(async () =>
{
    await Task.Delay(1000);
    semaphore.Release(); // no prior Wait() needed
});

await Task.WhenAll(consumer, producer);
```

> [!NOTE]
> `Release()` does NOT need to be preceded by `Wait()` — it simply increments the counter.
> Unlike `Monitor` and `Mutex`, calling `Release()` without a prior `Wait()` does not throw.

### Thread Affinity

**Thread affinity** means the same thread that acquired a primitive must release it.

| Primitive | Thread affinity | Works with async/await |
|---|---|---|
| `lock` / `Monitor` | ✅ Yes | ❌ No |
| `Mutex` | ✅ Yes | ❌ No |
| `SemaphoreSlim` | ❌ No | ✅ Yes |

`SemaphoreSlim` has no ownership tracking — any thread can call `Release()`. This is why it works with `async/await`: the continuation may resume on a different thread, but that doesn't matter.

### lock vs SemaphoreSlim(1,1)

Use `SemaphoreSlim(1, 1)` as a drop-in async replacement for `lock`:

```csharp
// Instead of: lock (_lock) { ... }
await _semaphore.WaitAsync();
try { /* work */ }
finally { _semaphore.Release(); }
```

### Comparison with other primitives

| | `lock` | `Monitor` | `Mutex` | `SemaphoreSlim` |
|---|---|---|---|---|
| Max concurrent | 1 | 1 | 1 | N |
| Async support | ❌ | ❌ | ❌ | ✅ |
| Cross-process | ❌ | ❌ | ✅ | ❌ |
| Thread affinity | ✅ | ✅ | ✅ | ❌ |
| Release before Wait | throws | throws | throws | ✅ allowed |


---

## ReaderWriterLockSlim

#threading #rwlock #caching

`ReaderWriterLockSlim` optimises for workloads where reads vastly outnumber writes. A plain `lock` serialises every operation — even two concurrent reads block each other. `ReaderWriterLockSlim` allows unlimited concurrent readers; only writes require exclusivity.

### When to use

- Shared in-memory cache read by many threads, refreshed rarely
- Configuration object reloaded on file change
- Any read-heavy, write-rare shared state

> [!NOTE]
> For write-heavy workloads, `ReaderWriterLockSlim` adds overhead vs `lock` with no benefit. Profile before reaching for it.

---

### Three lock modes

| Mode | Enter | Exit | Concurrency |
|---|---|---|---|
| Read | `EnterReadLock()` | `ExitReadLock()` | Many threads simultaneously |
| Upgradeable read | `EnterUpgradeableReadLock()` | `ExitUpgradeableReadLock()` | One at a time; readers can coexist |
| Write | `EnterWriteLock()` | `ExitWriteLock()` | Fully exclusive |

**Coexistence matrix:**

| Lock held → | + Read | + Upgradeable read | + Write |
|---|---|---|---|
| **Read** | ✅ | ✅ | ❌ |
| **Upgradeable read** | ✅ | ❌ | ❌ |
| **Write** | ❌ | ❌ | ❌ |

While an upgradeable read lock is held but not yet upgraded, regular readers still proceed. The upgrade blocks new readers only when `EnterWriteLock()` is called — at that point it waits for existing readers to drain.

---

### The upgradeable read lock

**Why it exists — two problems it solves:**

**1. Deadlock prevention.** If a thread holds a plain read lock and calls `EnterWriteLock()`, it deadlocks immediately: the write lock waits for all readers to finish, including itself. The thread is waiting for itself to release — it never will.

**2. Atomicity.** Without an upgradeable read, another writer could mutate the resource between your read and your write, invalidating what you just read.

```csharp
// WRONG — deadlocks immediately
_lock.EnterReadLock();
// ...read...
_lock.EnterWriteLock(); // waits for all readers including THIS thread → deadlock

// CORRECT
_lock.EnterUpgradeableReadLock();
try
{
    // read and check condition
    if (needsUpdate)
    {
        _lock.EnterWriteLock();
        try { /* mutate */ }
        finally { _lock.ExitWriteLock(); }
    }
}
finally { _lock.ExitUpgradeableReadLock(); }
```

> [!WARNING]
> Only **one thread** can hold the upgradeable read lock at a time. Two threads both trying to "read then maybe write" will serialize on this lock even if neither ends up writing.

---

### Full example — cached user store

```csharp
public class CachedUserStore : IDisposable
{
    private readonly ReaderWriterLockSlim _lock = new();
    private readonly Dictionary<int, string> _cache = new();

    public string? GetUser(int id)
    {
        _lock.EnterReadLock();
        try
        {
            _cache.TryGetValue(id, out var name);
            return name;
        }
        finally { _lock.ExitReadLock(); }
    }

    public void AddOrRefresh(int id, string name)
    {
        _lock.EnterUpgradeableReadLock();
        try
        {
            // Skip write if value is unchanged
            if (_cache.TryGetValue(id, out var existing) && existing == name)
                return;

            _lock.EnterWriteLock();
            try { _cache[id] = name; }
            finally { _lock.ExitWriteLock(); }
        }
        finally { _lock.ExitUpgradeableReadLock(); }
    }

    public void Remove(int id)
    {
        _lock.EnterWriteLock();
        try { _cache.Remove(id); }
        finally { _lock.ExitWriteLock(); }
    }

    // Any class owning an IDisposable field should implement IDisposable
    public void Dispose() => _lock.Dispose();
}
```

> [!WARNING]
> Never call `_lock.Dispose()` inside a method. The lock is shared state for the lifetime of the object — dispose it only when the object itself is disposed.

---

### Common mistakes

| Mistake | Consequence |
|---|---|
| `EnterReadLock()` then `EnterWriteLock()` | Deadlock — thread waits for itself |
| Using `EnterReadLock()` instead of `EnterWriteLock()` inside upgradeable block | Writing under a read lock — data corruption under concurrent reads |
| `Dispose()` inside a single method | All subsequent lock calls throw `ObjectDisposedException` |
| Two threads both using upgradeable read for "read then maybe write" | Unnecessary serialization — only one can hold it at a time |

---

### Comparison with other primitives

| | `lock` | `Mutex` | `SemaphoreSlim(1,1)` | `ReaderWriterLockSlim` |
|---|---|---|---|---|
| Concurrent readers | ❌ | ❌ | ❌ | ✅ |
| Async support | ❌ | ❌ | ✅ | ❌ |
| Cross-process | ❌ | ✅ | ❌ | ❌ |
| Upgrade read→write | ❌ | ❌ | ❌ | ✅ |
| Overhead | Low | High | Low | Medium |

> [!NOTE]
> `ReaderWriterLockSlim` has no `WaitAsync()` — it is **not async-compatible**. For async read-heavy scenarios, consider `SemaphoreSlim` or a dedicated async-friendly reader-writer library.

---

### Interview questions

**Q: Why can't you upgrade a plain read lock to a write lock?**
A: `EnterWriteLock()` waits for all current readers to exit. If the calling thread holds a read lock, it is one of those readers — it creates an unresolvable wait on itself.

**Q: Why is only one upgradeable read lock allowed at a time?**
A: If two threads both held upgradeable reads and both tried to upgrade, each would wait for the other to release before acquiring the write lock — a deadlock between upgraders. The single-slot constraint prevents this.

**Q: When is `ReaderWriterLockSlim` NOT worth it?**
A: When writes are frequent, or when the protected operation is so fast that the extra overhead of RWLS exceeds the gain from concurrent reads. Measure first.

---

---

## ManualResetEventSlim

#manualreseteventslim

`ManualResetEventSlim` is a lightweight signaling primitive that acts as a **gate**: it is either open (set) or closed (unset). All threads calling `Wait()` block when the gate is closed and are released simultaneously when it is opened.

> [!NOTE]
> `ManualResetEventSlim` is the lightweight managed alternative to `ManualResetEvent`. It uses spin-waiting before falling back to a kernel wait handle, making it faster for short waits.

### States and methods

| Method | What it does |
|---|---|
| `Set()` | Opens the gate — all waiting threads are released, future `Wait()` calls pass through instantly |
| `Reset()` | Closes the gate — future `Wait()` calls will block again |
| `Wait()` | Blocks the calling thread until the event is in set state |
| `Wait(timeout)` | Returns `bool` — `true` if set in time, `false` if timed out |
| `Wait(cancellationToken)` | Throws `OperationCanceledException` if token is cancelled |
| `Wait(timeout, cancellationToken)` | Combines both |

### Gate behaviour

```csharp
var mre = new ManualResetEventSlim(false); // starts closed

// Thread A — blocks
mre.Wait();

// Thread B — opens gate, releases ALL waiting threads
mre.Set();

// Thread C — arrives after Set(), passes through immediately
mre.Wait(); // does not block

// Reset closes the gate again
mre.Reset();
mre.Wait(); // blocks again
```

> [!WARNING]
> Unlike `AutoResetEvent`, the gate stays open after `Set()`. You must explicitly call `Reset()` to close it again.

### ManualResetEventSlim vs AutoResetEvent

| | `ManualResetEventSlim` | `AutoResetEvent` |
|---|---|---|
| Analogy | Door | Turnstile |
| Threads released per `Set()` | All waiting threads | Exactly one |
| Gate after `Set()` | Stays open | Closes immediately |
| Reset | Manual (`Reset()`) | Automatic |

### Typical pattern — startup barrier

```csharp
private readonly ManualResetEventSlim _ready = new ManualResetEventSlim(false);

// Loader thread
void Initialize()
{
    _ready.Reset();              // close gate before reload
    // ... heavy initialization ...
    _ready.Set();                // open gate — all workers proceed
}

// Worker threads
void DoWork(CancellationToken ct)
{
    bool canPass = _ready.Wait(3000, ct);  // timeout + cancellation

    if (!canPass)
        Console.WriteLine("Warning: config not ready in time, skipping.");
    else
        Console.WriteLine("Config ready — doing work.");
}
```

> [!NOTE]
> Call `Reset()` at the **start** of `Initialize()`, not at the end. Workers arriving mid-reload must be held back immediately — resetting after the load is too late.

### Async limitation

`ManualResetEventSlim.Wait()` is a **synchronous blocking call**. It works correctly in sync code and short-lived tools, but in ASP.NET Core it blocks a thread pool thread for the duration of the wait.

| | Blocks thread | Async-safe |
|---|---|---|
| `mre.Wait()` | ✅ | ⚠️ avoid in async server code |
| `semaphore.WaitAsync()` | ❌ | ✅ |

For broadcast signaling in async code, `TaskCompletionSource` is the common alternative.

### Interview questions

**Q: What is the difference between `ManualResetEventSlim` and `AutoResetEvent`?**
A: `ManualResetEventSlim` releases all waiting threads when `Set()` is called and stays open until `Reset()` is called explicitly. `AutoResetEvent` releases exactly one thread per `Set()` and resets itself automatically — it is a turnstile, not a door.

**Q: What happens if a thread calls `Wait()` after `Set()` but before `Reset()`?**
A: It passes through immediately without blocking. The gate is open and remains open until `Reset()` is called.

**Q: Why must `Reset()` be called at the start of a reload, not the end?**
A: Workers may arrive and call `Wait()` at any point during the reload. If the gate is still open from the previous load, they pass through and use a partially initialized resource. `Reset()` at the start closes the gate before any work begins, ensuring all workers wait for the full new initialization.

**Q: Why is `ManualResetEventSlim` not suitable as a drop-in in async server code?**
A: `Wait()` has no async overload — it blocks the calling thread. In ASP.NET Core under load this wastes thread pool threads. Use `TaskCompletionSource` for one-shot async signals instead.

---

## See also

- [[lock]]
- [[Monitor]]
- [[Mutex]]
- [[SemaphoreSlim]]
- [[ReaderWriterLockSlim]]
- [[ManualResetEventSlim]]
- [[05-synchronization-primitives]]
- [[02-async-await-internals]]
