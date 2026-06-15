---
tags: [phase2, threading, deadlock, race-condition, livelock, starvation, csharp]
created: 2026-06-15
phase: 2
topic: 7
---

# T7 — Deadlocks & Race Conditions

[[#Race Conditions]] | [[#Deadlocks]] | [[#Async Deadlock]] | [[#Livelock & Starvation]] | [[#Mental Checklist]]

---

## Race Conditions

A **race condition** occurs when two or more threads access shared state concurrently and the outcome depends on scheduling order.

> [!WARNING]
> `_counter++` is NOT atomic. It compiles to READ → ADD → WRITE (three CPU instructions).
> Two threads can both read the same stale value → both write the same result → one increment silently lost. This is called a **lost update**.

### Visibility race

Without a memory barrier the JIT can hoist a field into a CPU register and never re-read memory:

```csharp
bool _running = true;

// Thread A — JIT may cache _running in e.g. R15 register
while (_running) { }   // spins forever even after Thread B writes false

// Thread B
_running = false;      // writes to memory — Thread A never sees it
```

Observed in VS debugger: Locals shows "variable not available" (optimized away), Registers shows `R15 = 0x0000000000000001` (stale `true`).

### Fixes

```csharp
// 1. Interlocked — atomic read+add+write via CAS
Interlocked.Increment(ref _counter);

// 2. lock — mutual exclusion for compound operations
private readonly object _sync = new();
lock (_sync) { _counter++; }

// 3. volatile — memory visibility only (single read/write, NOT compound ops)
private volatile bool _running;

// 4. CancellationToken — idiomatic .NET for stop signals
while (!ct.IsCancellationRequested) { }
```

> [!WARNING]
> `volatile` does NOT give atomicity. `volatile int _counter; _counter++` is still a race condition.
> `volatile` fixes visibility (register hoisting). `Interlocked` fixes atomicity (lost updates). They solve different problems.

> [!NOTE]
> `volatile` is only valid on **fields**, not local variables or `ref` parameters.
> `lock` requires a **reference type** — you cannot `lock (_counter)` on an int.
> Always use a dedicated `private readonly object _sync = new()` as the lock target. `readonly` ensures the same object is always used.

---

## Deadlocks

A **deadlock** occurs when two or more threads are permanently blocked, each waiting on a lock held by the other. CPU usage: **0%** — threads are suspended by the OS.

### Coffman's four necessary conditions

ALL four must hold simultaneously. Breaking any one prevents the deadlock.

| # | Condition | Description | Break how |
|---|-----------|-------------|-----------|
| 1 | **Mutual exclusion** | Only one thread holds a lock at a time | Hard — defeats the purpose of locking |
| 2 | **Hold & Wait** | Thread holds ≥1 lock while waiting for another | Acquire all locks upfront (rarely practical) |
| 3 | **No preemption** | Locks can only be released voluntarily | `Monitor.TryEnter` with timeout |
| 4 | **Circular wait** | T1 waits for T2's lock, T2 waits for T1's lock | **Consistent lock ordering ← most practical** |

### Classic two-lock deadlock

```csharp
object lockA = new(), lockB = new();

// Thread A: lockA → lockB
lock (lockA) { Thread.Sleep(50); lock (lockB) { } }  // blocks waiting for lockB

// Thread B: lockB → lockA  ← ABBA pattern
lock (lockB) { Thread.Sleep(50); lock (lockA) { } }  // blocks waiting for lockA
```

In VS Parallel Stacks: both threads show ⊘ Deadlocked, each reporting "Waiting on lock owned by Thread X".

### Real-world disguise — Transfer example

The ABBA pattern can be hidden in business logic when callers control parameter order:

```csharp
// Thread A: Transfer(accountX, accountY)  → locks X then Y
// Thread B: Transfer(accountY, accountX)  → locks Y then X  ← deadlock!
```

Fix — impose consistent order inside the method using a stable tiebreaker (ID):

```csharp
void Transfer(Account from, Account to, decimal amount)
{
    var first  = from.Id < to.Id ? from : to;
    var second = from.Id < to.Id ? to   : from;

    lock (first)
    lock (second)
    {
        from.Balance -= amount;
        to.Balance   += amount;
    }
}
```

### Prevention strategies

```csharp
// 1. Consistent lock ordering (most common)
lock (lockA) { lock (lockB) { /* work */ } }  // always A before B, everywhere

// 2. Monitor.TryEnter with timeout — back off instead of blocking forever
bool got = Monitor.TryEnter(lockB, TimeSpan.FromSeconds(1));
if (!got) return;  // back off, retry later
try { /* work */ }
finally { Monitor.Exit(lockB); }

// 3. Lock-free — ConcurrentDictionary, ConcurrentQueue, Interlocked
// Can't deadlock if there are no locks

// 4. SemaphoreSlim — async-friendly, supports await
await _semaphore.WaitAsync(ct);
try   { /* work */ }
finally { _semaphore.Release(); }
```

---

## Async Deadlock

> [!WARNING]
> Never call `.Result` or `.Wait()` on a Task in UI or old ASP.NET contexts.

### Exact mechanism

1. `await` inside an async method captures the `SynchronizationContext` (e.g. WinForms UI thread)
2. When the async operation completes, runtime calls `context.Post(continuation)` — targeting the UI thread specifically
3. But the UI thread is blocked on `.Result` waiting for the Task
4. Continuation waits for the UI thread → UI thread waits for the continuation → **circular wait → deadlock**

```csharp
// ❌ Deadlock in WinForms
void Button_Click(object s, EventArgs e)
{
    var data = GetDataAsync().Result;  // blocks UI thread
    // continuation inside GetDataAsync tries to resume on UI thread
    // UI thread is blocked → deadlock
}

// ✅ Fix A: async all the way
async void Button_Click(object s, EventArgs e)
{
    var data = await GetDataAsync();
}

// ✅ Fix B: ConfigureAwait(false) in library code
// "don't capture SynchronizationContext — continuation can run on any ThreadPool thread"
async Task<string> GetDataAsync()
{
    var data = await http.GetStringAsync(url).ConfigureAwait(false);
    return data;
}
```

> [!NOTE]
> `ConfigureAwait(false)` doesn't mean "run on any thread". It means "don't capture the SynchronizationContext". The continuation won't target the UI thread, so the `.Result` block doesn't cause a cycle.

---

## Livelock & Starvation

### Livelock vs Deadlock

| | Deadlock | Livelock |
|---|---|---|
| Threads executing? | ❌ No — suspended by OS | ✅ Yes — running constantly |
| CPU usage | ~0% | High |
| Waiting on | Each other's lock | Nothing — keep retrying |
| Analogy | Two trains facing on one track | Two people in a corridor stepping the same way forever |

Livelock pattern — threads actively respond to each other and cancel each other out:

```csharp
while (true)
{
    if (otherThreadIsUsing) { stepAside = true; Thread.Sleep(10); continue; }
    stepAside = false;
    if (otherThreadIsUsing) { stepAside = true; continue; }  // conflict again
    DoWork();  // never reached
}
```

Fix: randomise back-off interval so threads don't retry in sync:
```csharp
Thread.Sleep(Random.Shared.Next(0, 50));
```

### Starvation

A thread is **indefinitely denied** access — not just slow, but never gets a turn.

Classic case with `ReaderWriterLockSlim`: multiple concurrent readers can hold the lock simultaneously, but a writer needs exclusive access and must wait for **all** readers to finish. If new readers arrive continuously, the writer never gets a window.

```csharp
// Mitigation 1: TryEnterWriteLock with timeout
if (!_rwLock.TryEnterWriteLock(TimeSpan.FromSeconds(2)))
    throw new TimeoutException("Could not acquire write lock");

// Mitigation 2: limit reader concurrency with SemaphoreSlim
// so a continuous flood of readers can't block writers indefinitely

// Mitigation 3: redesign with Channel<T> or ConcurrentQueue
// if write load is high — avoid the lock entirely
```

> [!NOTE]
> `ReaderWriterLockSlim` has partial fairness built in — once a writer is queued, new readers are blocked from entering. Pure starvation requires very high continuous reader load.

---

## Mental Checklist

- [ ] Shared state? → protect it
- [ ] `i++` on shared field? → NOT atomic → use `Interlocked` or `lock`
- [ ] Visibility only (flag/signal)? → `volatile` or `CancellationToken`
- [ ] Multiple locks? → enforce consistent acquisition order
- [ ] Blocking on Task (`.Result` / `.Wait()`)? → `await` all the way instead
- [ ] Library code with `await`? → add `ConfigureAwait(false)` to every await
- [ ] Long critical section? → minimise time inside lock
- [ ] Need timeout safety? → `Monitor.TryEnter` with timeout
- [ ] Can go lock-free? → prefer `Concurrent*` collections

---

## Debugging in VS

| Tool | How to open | What to look for |
|---|---|---|
| Parallel Stacks | Debug → Windows → Parallel Stacks | ⊘ icon = deadlocked thread, "Waiting on lock owned by Thread X" |
| Threads | Debug → Windows → Threads | Which thread is at which method, switch context by double-clicking |
| Call Stack | Debug → Windows → Call Stack | Enable "Show External Code" to see full frames |
| Registers | Debug → Windows → Registers | Hoisted flag visible as stale value in e.g. R15 |
| Locals | Automatic when paused | "Variable not available — module optimized" = JIT hoisted it to register |

> [!NOTE]
> Threads / Parallel Stacks are only available while paused (breakpoint or Break All = Ctrl+Alt+Break / Debug → Break All).
> Attaching a debugger inserts memory barriers — visibility bugs may not reproduce under the debugger. Run Release `.exe` directly to observe register hoisting reliably.

---

## See also

[[05-synchronization-primitives]] | [[06-concurrent-collections]]
