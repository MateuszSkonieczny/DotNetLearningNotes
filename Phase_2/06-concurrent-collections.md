---
created: 2025-06-14
tags: [csharp, threading, concurrent-collections, phase-2]
phase: 2
topic: 6
---

# Topic 6 — Concurrent Collections

[[Phase_2/05-synchronization-primitives|← Topic 5]] | [[Phase_2/07-deadlocks-race-conditions|Topic 7 →]]

## Table of Contents

- [[#Overview]]
- [[#IProducerConsumerCollection]]
- [[#ConcurrentQueue]]
- [[#ConcurrentStack]]
- [[#ConcurrentBag]]
- [[#ConcurrentDictionary]]
- [[#BlockingCollection]]
- [[#Comparison]]
- [[#Key Gotchas]]

---

## Overview

Concurrent collections live in `System.Collections.Concurrent`. They differ from normal collections by providing built-in synchronization — no external `lock` needed for individual operations.

```
IProducerConsumerCollection<T>
    ├── ConcurrentQueue<T>    FIFO · lock-free
    ├── ConcurrentStack<T>    LIFO · lock-free
    └── ConcurrentBag<T>      Unordered · thread-local deques

BlockingCollection<T>         Wrapper over IProducerConsumerCollection
ConcurrentDictionary<K,V>     Independent · implements IDictionary
```

> [!NOTE]
> `BlockingCollection<T>` is a **wrapper**, not a collection itself. `ConcurrentDictionary` is completely separate from the `IProducerConsumerCollection` hierarchy.

---

## IProducerConsumerCollection

The shared interface implemented by Queue, Stack, and Bag. Defines the contract `BlockingCollection` wraps.

| Member | Notes |
|---|---|
| `TryAdd(T)` | Thread-safe add |
| `TryTake(out T)` | Thread-safe remove — ordering depends on implementation |
| `ToArray()` | Thread-safe snapshot |
| `CopyTo(T[], index)` | Thread-safe snapshot copy |

The practical value: you can swap the backing store of `BlockingCollection` by passing any `IProducerConsumerCollection` to its constructor.

---

## ConcurrentQueue

**FIFO · lock-free · segment-based**

### Internals

Linked list of fixed-size array segments (~32 slots each). Both `Enqueue` and `TryDequeue` use `Interlocked` CAS — no lock held. When a segment fills, a new one is allocated and linked atomically.

A brief spin-wait can occur on `TryDequeue` if a slot has been claimed by a producer (CAS succeeded) but not yet written. The dequeuer spins on a sequence number until the write completes — this window is nanosecond-range.

### Key API

| Member | Notes |
|---|---|
| `Enqueue(item)` | Always succeeds — unbounded |
| `TryDequeue(out T)` | Returns `false` if empty; never blocks |
| `TryPeek(out T)` | Reads head without removing; snapshot |
| `IsEmpty` | Preferred over `Count == 0` |
| `Count` | O(segments) walk — snapshot, may be stale |

### Pattern

```csharp
var queue = new ConcurrentQueue<WorkItem>();

// producer(s)
queue.Enqueue(item);

// consumer(s)
while (!ct.IsCancellationRequested)
{
    if (queue.TryDequeue(out var item))
        Process(item);
    else
        await Task.Delay(10, ct); // avoid spin-burn
}
```

> [!WARNING]
> `if (!q.IsEmpty) q.TryDequeue(out var x)` is a TOCTOU race. Always act on the `bool` from `TryDequeue` directly.

### When to use

Multi-producer / multi-consumer pipelines needing FIFO without blocking or backpressure. When you need blocking → `BlockingCollection`.

---

## ConcurrentStack

**LIFO · lock-free · linked nodes**

### Internals

Singly-linked list. The head pointer is a single `volatile` reference. Push and pop both CAS directly on the head — simpler than Queue's segment management. No spin-wait issue because a node is fully constructed before it's ever linked in.

### Key differences vs ConcurrentQueue

| | ConcurrentQueue | ConcurrentStack |
|---|---|---|
| Ordering | FIFO | LIFO |
| Structure | Linked segments | Linked individual nodes |
| Spin-wait on dequeue | Yes | No |
| Bulk operations | None | `PushRange` / `TryPopRange` |
| GC pressure | Low (amortised) | Higher (one alloc per item) |

### Key API

| Member | Notes |
|---|---|
| `Push(item)` | Always succeeds — unbounded |
| `TryPop(out T)` | Returns `false` if empty |
| `PushRange(T[])` | Atomically pushes entire array |
| `TryPopRange(T[])` | Atomically pops up to N items; returns count popped |

### Pattern — batch undo

```csharp
var history = new ConcurrentStack<string>();

// push batch atomically
history.PushRange(new[] { "action1", "action2", "action3" });

// pop up to 5 atomically
var buffer = new string[5];
int popped = history.TryPopRange(buffer);
```

### When to use

Undo/redo stacks, DFS work queues, LIFO processing. Also as backing store for `BlockingCollection` when LIFO + blocking is needed.

---

## ConcurrentBag

**Unordered · thread-local deques · work stealing**

### Internals

Each thread gets its own private deque, allocated on first use (not pre-divided). `Add` and `TryTake` on the **same thread** hit the thread-local deque — no lock, no contention. When a thread's deque is empty and it calls `TryTake`, it **steals** from another thread's deque — requires a lock on that deque.

```
Thread A's deque:  ← steal end | obj1 | obj2 | obj3 | local end →

Add/TryTake (same thread) → local end → fast, no lock
TryTake (different thread) → steal end → slow, acquires lock
```

### Key API

| Member | Notes |
|---|---|
| `Add(item)` | Adds to calling thread's local deque |
| `TryTake(out T)` | Takes from local deque (LIFO-ish); steals if empty |
| `TryPeek(out T)` | Peeks local deque |
| `Count` | Acquires all thread locks — expensive |

### Pattern — object pool

```csharp
var pool = new ConcurrentBag<DbConnection>();

DbConnection Rent()
{
    if (pool.TryTake(out var conn)) return conn;
    return new DbConnection();
}

void Return(DbConnection conn) => pool.Add(conn);
```

> [!WARNING]
> Do NOT use for classic producer-consumer (one thread adds, different thread removes). Every `TryTake` becomes a steal → lock contention on every operation. Use `ConcurrentQueue` instead.

---

## ConcurrentDictionary

**Key-value · striped locks · IDictionary**

### Internals

Hash table divided into stripes (buckets), each with its own lock. Two threads writing to different stripes never contend. Default stripe count = processor count. Single-key operations are atomic; multi-key operations are not.

### Key API

| Member | Atomic? | Notes |
|---|---|---|
| `TryAdd(k, v)` | Yes | Adds only if key absent |
| `TryGetValue(k, out v)` | Yes | Safe read |
| `TryUpdate(k, newV, compV)` | Yes | Updates only if current == `compV` — CAS |
| `TryRemove(k, out v)` | Yes | Removes and returns old value |
| `GetOrAdd(k, factory)` | Partial | Factory may run multiple times |
| `AddOrUpdate(k, addV, updateFn)` | Partial | Update factory may run multiple times |

### Atomic counter

```csharp
var hits = new ConcurrentDictionary<string, int>();
hits.AddOrUpdate("endpoint", 1, (_, old) => old + 1);
```

### Safe lazy cache with Lazy\<T\>

`GetOrAdd` does not call the factory under a lock — the factory (the lambda that produces the value) can run multiple times if threads race. Fix: use `Lazy<T>` as the value type.

```csharp
var cache = new ConcurrentDictionary<string, Lazy<string>>();

var lazy = cache.GetOrAdd(key, k => new Lazy<string>(() => ExpensiveLoad(k)));
var value = lazy.Value; // runs at most once per stored Lazy instance
```

**Why this works:**
- `GetOrAdd` may create multiple `Lazy` instances, but only one gets stored ("first writer wins")
- All threads receive the same stored `Lazy` instance back from `GetOrAdd`
- All threads call `.Value` on the **same object** → `Lazy<T>` guarantees single execution per instance

> [!WARNING]
> Multi-key atomicity is not possible. To atomically update two keys you need an external lock around both operations.

---

## BlockingCollection

**Blocking wrapper · optional bound · clean shutdown**

### What it is

Not a collection — a wrapper over any `IProducerConsumerCollection<T>` that adds:
1. **Blocking `Take`** — consumer waits instead of spinning when empty
2. **Bounded capacity** — producer blocks when full (backpressure)
3. **`CompleteAdding()`** — clean shutdown signal

Default backing store: `ConcurrentQueue<T>`.

> [!NOTE]
> `BlockingCollection` blocks threads synchronously via `SemaphoreSlim.Wait()`. The thread is parked by the OS — it doesn't burn CPU but it is **not** returned to the ThreadPool. For async pipelines use `Channel<T>` (Topic 9).

### Key API

| Member | Notes |
|---|---|
| `Add(item)` | Blocks if at bounded capacity; throws if completed |
| `TryAdd(item, timeout)` | Non-blocking variant |
| `Take()` | Blocks until item available |
| `TryTake(out T, timeout)` | Non-blocking variant |
| `CompleteAdding()` | Irreversible — signals no more items coming |
| `GetConsumingEnumerable()` | Blocking foreach — exits when completed AND empty |
| `IsCompleted` | `true` when completed and empty |

### Pattern — bounded pipeline

```csharp
var bc = new BlockingCollection<WorkItem>(boundedCapacity: 100);

// producer
Task.Run(() =>
{
    foreach (var item in GetItems())
        bc.Add(item); // blocks if full
    bc.CompleteAdding();
});

// consumer
foreach (var item in bc.GetConsumingEnumerable())
    Process(item); // exits cleanly when completed + empty
```

### Safe shutdown with multiple producers

`CompleteAdding()` must be called **exactly once**, after **all** producers are done:

```csharp
await Task.WhenAll(producer1, producer2);
bc.CompleteAdding(); // safe — both producers finished
await consumer;
```

Calling it inside one producer risks the other producer throwing `InvalidOperationException` on its next `Add`.

### Swapping the backing store

```csharp
// LIFO
var bc = new BlockingCollection<T>(new ConcurrentStack<T>(), boundedCapacity: 50);

// Unordered
var bc = new BlockingCollection<T>(new ConcurrentBag<T>());
```

---

## Comparison

| Collection | Order | Lock-free | Blocks | Bounded |
|---|---|---|---|---|
| `ConcurrentQueue` | FIFO | Yes | No | No |
| `ConcurrentStack` | LIFO | Yes | No | No |
| `ConcurrentBag` | None | Partial | No | No |
| `ConcurrentDictionary` | N/A | Striped | No | No |
| `BlockingCollection` | Backing store | No | Yes | Optional |

### Decision guide

| Need | Use |
|---|---|
| FIFO, non-blocking | `ConcurrentQueue` |
| FIFO + blocking consumer | `BlockingCollection` (default) |
| FIFO + backpressure | `BlockingCollection` with capacity |
| LIFO | `ConcurrentStack` |
| LIFO + blocking | `BlockingCollection` wrapping `ConcurrentStack` |
| Shared cache / lookup | `ConcurrentDictionary` |
| Per-key atomic counter | `ConcurrentDictionary` + `AddOrUpdate` |
| Object pool | `ConcurrentBag` |
| Async producer-consumer | `Channel<T>` (Topic 9) |

---

## Key Gotchas

| Trap | Collection | Fix |
|---|---|---|
| TOCTOU on `IsEmpty` check | `ConcurrentQueue` | Act on `TryDequeue` bool directly |
| Factory runs multiple times | `ConcurrentDictionary` | Use `Lazy<T>` as value type |
| Multi-key not atomic | `ConcurrentDictionary` | External lock around both ops |
| Work-steal contention | `ConcurrentBag` | Switch to `ConcurrentQueue` for prod-consumer |
| `Add` after `CompleteAdding` | `BlockingCollection` | Call `CompleteAdding` once, after all producers |
| Thread blocked, not freed | `BlockingCollection` | Use `Channel<T>` for async scenarios |
| GC pressure under high load | `ConcurrentStack` | Use `ConcurrentQueue` (amortised segments) |

---

## See also

- [[Phase_2/05-synchronization-primitives|Topic 5 — Synchronization Primitives]]
- [[Phase_2/07-deadlocks-race-conditions|Topic 7 — Deadlocks & Race Conditions]]
- [[Phase_2/09-channels-dataflow|Topic 9 — Channels & Dataflow]]
