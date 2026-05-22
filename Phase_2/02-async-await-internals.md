---
tags: [dotnet, csharp, async, threading, phase2, interview]
created: 2026-05-22
phase: 2
topic: 2
status: complete
---

# async/await Internals

<!-- TOC -->
- [[#State Machine Generation]]
- [[#State Machine Fields]]
- [[#MoveNext — How It Works]]
- [[#Execution Flow]]
- [[#SynchronizationContext]]
- [[#ConfigureAwait(false)]]
- [[#The Classic Deadlock]]
- [[#async void vs async Task]]
- [[#Task.Yield]]
- [[#ValueTask]]
- [[#Common Pitfalls]]
- [[#Interview Answers]]
- [[#See also]]
<!-- /TOC -->

---

## State Machine Generation

`async`/`await` is **pure compiler sugar** (Roslyn). Every `async` method is transformed into a **private state machine struct** implementing `IAsyncStateMachine`. You never see this in source code — it exists only in the compiled output.

```csharp
// Your code
public async Task<string> GetDataAsync()
{
    var response = await httpClient.GetStringAsync(url);
    return response;
}

// What Roslyn generates (simplified)
private struct GetDataAsync_StateMachine : IAsyncStateMachine
{
    public int _state;                                    // -1, 0, 1...
    public AsyncTaskMethodBuilder<string> _builder;       // manages the outer Task
    private TaskAwaiter<string> _awaiter;                 // for the current await

    void IAsyncStateMachine.MoveNext() { ... }
}
```

> [!NOTE]
> No threads are created. No magic runtime. Just a struct + a callback.

---

## State Machine Fields

| Field | Purpose |
|---|---|
| `_state` | Tracks where the method is. -1 = not started. 0..N = suspended at await N |
| `_builder` | Manages the outer Task. Registers callbacks. Sets final return value |
| `_awaiter1`, `_awaiter2`... | One per await. Holds the awaiter for each async operation |
| `_x`, `_y`... | Fields for local variables that need to survive across awaits |

### _builder has two responsibilities

1. **Register callbacks** — `AwaitUnsafeOnCompleted(ref _awaiter, ref this)` — tells the runtime: "when this awaiter finishes, call MoveNext() on this state machine"
2. **Complete the outer Task** — `SetResult(value)` at the final return, `SetException(ex)` if an exception occurs

> [!NOTE]
> `_builder` does NOT hold intermediate await results. Each `_awaiter` holds its own result. `_builder` only cares about the final `return` value of the whole method.

---

## MoveNext — How It Works

Each `await` becomes one `case` in a `switch` statement. The pattern is identical every time:

```csharp
void MoveNext()
{
    switch (_state)
    {
        case -1:  // first call — not a resume target, just the initial state
            _awaiter1 = httpClient.GetStringAsync(url).GetAwaiter();
            if (!_awaiter1.IsCompleted)          // slow path: not done yet
            {
                _state = 0;                      // bookmark: "suspended at await #1"
                _builder.AwaitUnsafeOnCompleted(ref _awaiter1, ref this); // register callback
                return;                          // ← release the thread
            }
            goto case 0;                         // fast path: already done, skip suspension

        case 0:   // resumed after GetStringAsync finished
            string result = _awaiter1.GetResult();  // get value (or rethrow exception)
            _builder.SetResult(result);             // complete the outer Task
            break;
    }
}
```

### Pattern per case (except last)

```
case N:
  1. Start the next async operation
  2. Check IsCompleted
     → not done: save state, register callback, return (release thread)
     → done: fall through to next case (fast/synchronous path)
```

### Last case

```
  Get result from awaiter
  _builder.SetResult(finalValue)  ← completes the Task the caller is holding
```

### Multiple awaits example

```csharp
public async Task<int> ProcessAsync()
{
    await Task.Delay(100);          // await #1 → case -1, suspends to state 0
    int x = await GetValueAsync();  // await #2 → case 0,  suspends to state 1
    await Task.Delay(50);           // await #3 → case 1,  suspends to state 2
    return x * 2;                   // case 2 → _builder.SetResult(x * 2)
}
// Fields: _state, _builder, _awaiter1, _awaiter2, _awaiter3, _x
```

---

## Execution Flow

1. Caller invokes async method → state machine struct created, `MoveNext()` called (state = -1)
2. Hits `await` → calls `GetAwaiter()`, checks `IsCompleted`
3. **Fast path** (`IsCompleted == true`) → `GetResult()` immediately, no suspension, no allocation
4. **Slow path** (`IsCompleted == false`) → saves state, registers callback via `AwaitUnsafeOnCompleted`, calls `return` — **thread is released**
5. OS/driver signals I/O completion — no thread was waiting
6. Thread pool thread picks up the callback → calls `MoveNext()` again
7. Switch jumps to the saved state, execution continues
8. On final `return` → `_builder.SetResult(value)` completes the outer Task → caller's `await` unblocks

> [!NOTE]
> The thread is not blocked during I/O. `return` releases it completely. A different thread pool thread resumes the method later.

---

## SynchronizationContext

A `SynchronizationContext` is a scheduler that manages **on which thread async continuations resume after an await**.

When an `await` suspends, it captures the current `SynchronizationContext`. After the awaited task completes, the continuation is posted back to that context.

| Environment | SynchronizationContext | Continuation runs on |
|---|---|---|
| WinForms / WPF | ✅ exists | UI thread |
| MAUI / Blazor | ✅ exists | UI/dispatch thread |
| ASP.NET Classic | ✅ exists | Request thread |
| ASP.NET Core | ❌ none | Any thread pool thread |
| Console | ❌ none | Any thread pool thread |

---

## ConfigureAwait(false)

Tells the awaiter: **do not capture the SynchronizationContext**. The continuation runs on any available thread pool thread.

```csharp
await SomeWork().ConfigureAwait(false);  // don't post back to original context
```

| You're writing | Use ConfigureAwait(false)? |
|---|---|
| App code — awaiting correctly | No — doesn't matter |
| Library / NuGet package | **Yes — always** — you don't control the caller |
| WinForms/WPF app code touching UI | No — you want to return to UI thread |

> [!NOTE]
> In ASP.NET Core there is no SynchronizationContext, so `ConfigureAwait(false)` has no practical effect on app code. It still matters in library code because that library may be used in WinForms or MAUI apps.

---

## The Classic Deadlock

```csharp
// WinForms button handler
private void Button_Click(object sender, EventArgs e)
{
    string result = GetDataAsync().Result;  // ← blocks UI thread
    label.Text = result;
}

// Library method — no ConfigureAwait(false)
public async Task<string> GetDataAsync()
{
    return await httpClient.GetStringAsync(url);  // captures SyncCtx: "return to UI thread"
}
```

**Why it deadlocks:**
1. UI thread calls `.Result` → blocks itself
2. Continuation captured: "resume on UI thread after HTTP finishes"
3. HTTP finishes → continuation tries to post to UI thread
4. UI thread is blocked waiting for `.Result` → neither can proceed → **deadlock**

**Two fixes:**

```csharp
// Fix 1 — async all the way (preferred)
private async void Button_Click(object sender, EventArgs e)
{
    string result = await GetDataAsync();  // UI thread released, not blocked
    label.Text = result;
}

// Fix 2 — ConfigureAwait(false) in library code
public async Task<string> GetDataAsync()
{
    return await httpClient.GetStringAsync(url).ConfigureAwait(false);
    // continuation no longer needs UI thread → no deadlock even with .Result
}
```

> [!WARNING]
> Never call `.Result` or `.Wait()` on async code from a thread that has a SynchronizationContext.

---

## async void vs async Task

```csharp
// ✅ Always prefer async Task
public async Task DoWorkAsync()
{
    await Something();
}

// ⚠️ async void — only for event handlers
private async void Button_Click(object sender, EventArgs e)
{
    await DoWorkAsync();
}
```

| | async Task | async void |
|---|---|---|
| Can be awaited | ✅ yes | ❌ no |
| Exception propagation | via Task — observable | SynchronizationContext → may crash |
| Completion observable | ✅ yes | ❌ no |
| Use case | Everything | UI event handlers only |

**Why event handlers require void:** The framework defines the event handler signature as `void`. You cannot change it to `Task`. `async void` is the only way to use `await` inside an event handler — not because it's safe, but because there is no alternative.

The exception danger still exists even in event handlers — it's just accepted as unavoidable.

---

## Task.Yield

An `async` method that never hits a real `await` runs **synchronously**, hogging the thread the entire time:

```csharp
// Looks async — but never suspends → runs synchronously on calling thread
public async Task ProcessAsync()
{
    for (int i = 0; i < 1_000_000; i++)
        DoQuickMath(i);
}
```

`Task.Yield()` forces a suspension point — the method requeues itself at the back of the thread pool queue, letting other pending work run first:

```csharp
public async Task ProcessAsync()
{
    for (int i = 0; i < 1_000_000; i++)
    {
        if (i % 100 == 0)
            await Task.Yield();  // pause, let other work run, come back
        DoQuickMath(i);
    }
}
```

It is not a fixed delay — it means "go to the back of the line."

**When to use vs Task.Run:**

| Situation | Use |
|---|---|
| CPU work called from UI thread | `Task.Run` — offloads to thread pool |
| CPU work already on thread pool | `Task.Yield` — shares the thread, no extra overhead |
| Implementing an interface that can't use Task.Run | `Task.Yield` |

---

## ValueTask

`Task<T>` is a class — every call allocates on the heap. For hot-path code where the result is **often available synchronously** (e.g. cache hit), this allocation is wasted.

`ValueTask<T>` is a **struct** — when the result is synchronously available it wraps the value directly with zero heap allocation:

```csharp
public ValueTask<User> GetUserAsync(int id)
{
    if (_cache.TryGetValue(id, out var user))
        return new ValueTask<User>(user);           // sync path — no allocation ✅

    return new ValueTask<User>(LoadFromDbAsync(id)); // async path — falls back to Task
}
```

### Rules — treat ValueTask as a single-use ticket

- Await **exactly once**
- Await **immediately** — do not store it and await later
- Do **not** await after the producing method has returned
- Do **not** use when result is always async (just use `Task<T>`)

**Why:** A `ValueTask` may be backed by a pooled `IValueTaskSource` object. That object is recycled after one use. Awaiting twice or late means reading from a recycled object — undefined behavior, potentially wrong value.

```csharp
// ✅ correct
int result = await GetCachedValueAsync();

// ❌ dangerous — other work runs before await, backing object may be recycled
ValueTask<int> vt = GetCachedValueAsync();
await Task.Delay(100);
int result = await vt;
```

| | Task\<T\> | ValueTask\<T\> |
|---|---|---|
| Type | class (heap) | struct (stack) |
| Result often sync | allocates unnecessarily | ✅ no allocation |
| Always async I/O | ✅ simple, safe | overkill |
| Await multiple times | ✅ safe | ❌ undefined behavior |
| Default choice | ✅ yes | only when profiler shows need |

---

## Common Pitfalls

| Pitfall | Problem | Fix |
|---|---|---|
| `.Result` / `.Wait()` with SyncCtx | Deadlock | async all the way |
| `async void` outside event handlers | Unobservable exceptions, possible crash | Return `Task` |
| No `ConfigureAwait(false)` in library code | Caller's `.Result` can deadlock | Add `ConfigureAwait(false)` |
| `ValueTask` awaited twice | Undefined behavior | Await exactly once, immediately |
| `ValueTask` stored before await | Backing object may be recycled | Inline the await |
| `new Task(...)` | Cold task — never starts | Use `Task.Run()` or async method |
| CPU loop in async method | Hogs the thread, never yields | `Task.Yield()` or `Task.Run()` |

---

## Interview Answers

**"What does async/await compile to?"**
> The compiler generates a private state machine struct implementing `IAsyncStateMachine`. It has a `_state` int field, a `_builder` for managing the outer Task, an `_awaiter` per await call, and fields for local variables. `MoveNext()` contains a switch on `_state` — each case corresponds to one await. On suspension it saves the state, registers a callback via `AwaitUnsafeOnCompleted`, and returns — releasing the thread. When the awaited operation finishes, a thread pool thread calls `MoveNext()` again, the switch jumps to the saved state, and execution continues. At the final return, `_builder.SetResult()` completes the outer Task.

**"What is a SynchronizationContext?"**
> A scheduler that controls which thread async continuations resume on after an await. In WinForms/WPF/MAUI, it ensures continuations return to the UI thread. In ASP.NET Core there is none, so continuations run on any thread pool thread.

**"Explain the classic async deadlock."**
> UI thread calls `.Result` on an async method — blocking itself. The async method captures the SynchronizationContext: "resume on UI thread." The I/O finishes and the continuation tries to post back to the UI thread. But the UI thread is blocked waiting for `.Result`. Neither can proceed — deadlock. Fix: use `await` all the way, or `ConfigureAwait(false)` in library code.

**"When would you use ValueTask?"**
> When the result is often available synchronously — like a cache lookup — and the method is called very frequently. `ValueTask` is a struct so it avoids the heap allocation of `Task` on the synchronous path. The rule is: await it exactly once, immediately. Never store it or await it late — the backing object may be recycled.

---

## See also

- [[01-task-thread-threadpool]]
- [[03-cancellation-token]]
- [[05-task-combinators]]
