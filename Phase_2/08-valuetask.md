---
tags: [phase2, threading, async, valuetask, performance, csharp]
created: 2026-06-17
phase: 2
topic: 8
---

# T8 — ValueTask

[[#Why ValueTask Exists]] | [[#Internal Structure]] | [[#Task vs ValueTask]] | [[#The Core Rules]] | [[#The async Keyword Trap]] | [[#IValueTaskSource — Pooling]] | [[#IAsyncEnumerable & IAsyncDisposable]] | [[#When To Actually Use It]] | [[#Pitfalls]] | [[#Mental Checklist]]

---

## Why ValueTask Exists

`Task<T>` is a **heap-allocated reference type**. Every `async` method returning `Task<T>` allocates — even when it completes synchronously (cache hit, buffered read, short-circuit).

`ValueTask<T>` is a **struct** that can represent a synchronously-completed result **without any heap allocation**.

> [!NOTE]
> This is a targeted optimization for high-frequency, sync-completing hot paths — not a default replacement for `Task`. For typical service/app code with realistic call volume (e.g. tens of concurrent users), `Task` is simpler, safer, and the right choice. Gen0 GC is cheap; ValueTask only earns its complexity when allocations actually show up in a profiler.

---

## Internal Structure

`ValueTask<T>` wraps exactly one of three things:

| Inner state | Allocates? | When |
|---|---|---|
| `T` value directly | ❌ No | Sync completion, no `async` keyword |
| `Task<T>` | ✅ Yes (Task) | Async path fallback |
| `IValueTaskSource<T>` reference + token | ♻️ Pooled | Low-level libs (Socket, Pipe, Channel) |

```csharp
public ValueTask<int> GetAsync(int id)
{
    if (_cache.TryGetValue(id, out var value))
        return new ValueTask<int>(value);       // wraps T directly — no heap

    return new ValueTask<int>(LoadAsync(id));    // wraps Task<int> — heap, but expected (real I/O)
}

private async Task<int> LoadAsync(int id) { /* ... */ }
```

Non-generic `ValueTask` (no result value) uses `default` for the zero-alloc sync-complete case — there's no value to wrap, so the struct's zeroed state itself means "already completed successfully":

```csharp
public ValueTask FlushAsync()
{
    if (!_dirty)
        return default;                  // == new ValueTask() — both zero the same fields

    return new ValueTask(ActualFlushAsync());
}
```

---

## Task vs ValueTask

| | `Task<T>` | `ValueTask<T>` |
|---|---|---|
| Type | class (reference) | struct (value) |
| Sync-complete alloc | ❌ Yes | ✅ No |
| Async-complete alloc | ❌ Yes | ❌ Yes (Task fallback) |
| Await multiple times | ✅ Safe | ❌ Undefined behavior |
| `WhenAll` / `WhenAny` | ✅ Native | ❌ Needs `.AsTask()` |
| Safe to store in a field | ✅ Yes | ❌ No |
| Default choice | ✅ Yes | Only if measured |

---

## The Core Rules

> [!WARNING]
> **Await exactly once.** A `ValueTask` may be awaited at most one time. After that await completes, the instance is consumed — do not store it, do not await it again, do not share it across threads.

> [!WARNING]
> **Never store in a field.** Convert with `.AsTask()` first if you need to hold a reference across time.

> [!WARNING]
> **No combinators.** `Task.WhenAll` / `Task.WhenAny` have no `ValueTask` overload — always `.AsTask()` first.

`Task`, by contrast, is a reference type that never resets — once complete, reading its result twice is always safe because nothing about it changes between reads. The danger is specific to `ValueTask`'s pooled-source case, but the rule is applied universally because you can't tell from the outside which internal state (value / Task / pooled source) a given `ValueTask` is actually holding.

---

## The async Keyword Trap

The `async` keyword causes the compiler to generate a **state machine class** — a heap allocation — regardless of return type. The zero-alloc benefit of `ValueTask` **only applies when you skip `async` entirely** and return the struct directly.

```csharp
// ❌ async keyword → state machine still allocated, even on the sync path
public async ValueTask<int> FooAsync()
{
    if (_cache.TryGetValue(1, out var v)) return v;   // state machine still runs
    return await _db.LoadAsync(1);
}

// ✅ no async keyword → true zero-alloc on the sync path
public ValueTask<int> FooAsync()
{
    if (_cache.TryGetValue(1, out var v))
        return new ValueTask<int>(v);                 // pure struct
    return new ValueTask<int>(SlowAsync());
}
```

> [!NOTE]
> "ValueTask + async keyword" is a common false optimization — it looks correct but allocates exactly like `Task` would. The actual savings require manually returning the struct from a non-`async` method.

---

## IValueTaskSource — Pooling

For extremely hot paths (sockets, pipes processing millions of ops/sec), even one `Task` allocation per async completion matters. `IValueTaskSource<T>` lets a **single object be reused across many operations** instead of allocating a new completion object each time — the same conceptual pattern as renting from a `ConcurrentBag<T>` or an `ObjectPool<T>`: rent, use, reset, return.

```csharp
class MySource : IValueTaskSource<int>
{
    private ManualResetValueTaskSourceCore<int> _core;   // does the bookkeeping for you

    public ValueTask<int> GetValueTask()
        => new ValueTask<int>(this, _core.Version);      // version token = staleness check

    public int GetResult(short token) => _core.GetResult(token);
    public ValueTaskSourceStatus GetStatus(short token) => _core.GetStatus(token);
    public void OnCompleted(Action<object?> continuation, object? state, short token, ValueTaskSourceOnCompletedFlags flags)
        => _core.OnCompleted(continuation, state, token, flags);

    public void Complete(int result) => _core.SetResult(result);
    public void Reset() => _core.Reset();                // wipe + bump version → ready for reuse
}
```

`ManualResetValueTaskSourceCore<T>` is **not** part of `ValueTask` itself — it's an optional helper struct for *implementing* `IValueTaskSource<T>` correctly, used internally by `Socket`, `Pipe`, and `Channel`. `ValueTask` only knows how to call the four interface methods; it doesn't care how they're implemented underneath.

```
ValueTask<T>  (struct)
  ├─ T value           → no source
  ├─ Task<T>            → no source
  └─ IValueTaskSource<T> reference + token
              └─ implemented by your class, typically delegating to
                 ManualResetValueTaskSourceCore<T> internally
```

### Why double-await is dangerous here

```
Round 1: source handed out with version=5 → caller awaits, reads "first"
Reset(): source wiped, version bumped to 6 → ready for reuse
Round 2: same source handed out with version=6 → caller awaits, reads "second"

Stale re-await: old ValueTask (version=5) awaited again
  → ManualResetValueTaskSourceCore checks token → mismatch (5 ≠ 6)
  → throws InvalidOperationException (the SAFE failure mode)
```

> [!WARNING]
> .NET's own `ManualResetValueTaskSourceCore` validates the version token defensively and throws on mismatch — a clean failure. But a hand-rolled `IValueTaskSource` that skips token validation could silently return the **wrong value** with no exception at all. The language spec calls this "undefined behavior" precisely because it doesn't guarantee a specific outcome across all possible implementations — only that relying on it is unsupported.

---

## IAsyncEnumerable & IAsyncDisposable

These BCL interfaces are defined to return `ValueTask`, not by convention but by contract:

```csharp
interface IAsyncEnumerator<out T>
{
    ValueTask<bool> MoveNextAsync();   // called potentially millions of times in a loop
}

interface IAsyncDisposable
{
    ValueTask DisposeAsync();
}
```

`MoveNextAsync()` is called on every iteration of `await foreach`. If buffered data is ready, the call completes synchronously — `ValueTask<bool>` lets that common case skip allocation entirely. You don't need to manually manage this when writing `async IAsyncEnumerable<T>` with `yield return` — the compiler and runtime handle the ValueTask machinery for you.

---

## When To Actually Use It

| Scenario | Use ValueTask? |
|---|---|
| Frequently completes synchronously (cache, buffer) | ✅ Yes |
| Implementing `IAsyncEnumerable` / `IAsyncDisposable` | ✅ Yes (required by interface) |
| Low-level library (Socket, Pipe, Channel) | ✅ Yes |
| General business / service layer code | ❌ No — use `Task` |
| Result needs caching or multi-await | ❌ No — use `Task` |
| "Might be faster" without measurement | ⚠️ Measure first |

> [!NOTE]
> Interview-strong answer: knowing *when not to* use an optimization is as valuable as knowing the optimization. "Should you always use ValueTask instead of Task?" → No — it's a targeted fix for high-frequency sync-completion paths, not a default.

---

## Pitfalls

```csharp
// ❌ Double await
var vt = GetCachedAsync(42);
var a = await vt;   // ok
var b = await vt;   // undefined — source may be recycled
// FIX: var t = vt.AsTask(); await t; await t; // Task is safe to await repeatedly

// ❌ Storing in a field
class Foo { ValueTask<int> _pending; }   // dangerous
// FIX: Task<int> _pending = valueTask.AsTask();

// ❌ WhenAll with ValueTask
await Task.WhenAll(GetAsync(1), GetAsync(2));   // compile error — no overload
// FIX:
await Task.WhenAll(GetAsync(1).AsTask(), GetAsync(2).AsTask());
```

---

## Mental Checklist

- [ ] Does this method complete synchronously often (cache/buffer)? → candidate for `ValueTask`
- [ ] Did I drop the `async` keyword to actually get the zero-alloc benefit?
- [ ] Am I awaiting this `ValueTask` more than once anywhere? → convert with `.AsTask()` first
- [ ] Am I storing this `ValueTask` in a field? → convert with `.AsTask()` first
- [ ] Am I passing this to `WhenAll`/`WhenAny`? → convert with `.AsTask()` first
- [ ] Is this general app/service code with modest call volume? → just use `Task`, don't bother
- [ ] Implementing `IAsyncEnumerable`/`IAsyncDisposable`? → `ValueTask` is mandatory by interface signature

---

## See also

[[07-deadlocks-race-conditions]]
