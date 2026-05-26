---
tags: [phase-2, threading, async, cancellation, dotnet, csharp]
created: 2026-05-26
phase: 2
topic: 3
status: done
---

# CancellationToken

## Table of contents
- [[#Why it exists]]
- [[#The three types]]
- [[#Ownership and flow]]
- [[#How to observe cancellation]]
- [[#How cancellation works internally]]
- [[#Real-world patterns]]
- [[#Common traps]]
- [[#See also]]

---

## Why it exists

Async operations can take a long time, or they can be triggered and then become no longer needed before they finish. Without a cancellation mechanism, a method has no way to stop early — it runs to completion even if the result is thrown away.

> [!example]
> A user starts generating a large file. While the code is running they click Cancel. Without `CancellationToken` the method finishes generating the file anyway — wasting CPU, memory, and time. With `CancellationToken` the method can detect the signal and stop immediately.

`CancellationToken` is .NET's standard cooperative cancellation model. The operation is not killed forcefully — it checks for the signal at safe points and stops itself cleanly.

---

## The three types

| Type | Role | Notes |
|---|---|---|
| `CancellationTokenSource` | Controller — creates and owns the signal | Class, must be disposed |
| `CancellationToken` | Read-only view — passed into methods | Struct, cheap to copy |
| `CancellationToken.None` | Sentinel that never cancels | Safe default parameter |

```csharp
// Source — you own this
using var cts = new CancellationTokenSource();
cts.Cancel();                              // immediate
cts.CancelAfter(TimeSpan.FromSeconds(5)); // or timeout

// Token — pass this around
CancellationToken token = cts.Token;
```

---

## Ownership and flow

The method that **creates** the CTS owns it — cancels it, disposes it.
The method that **receives** the token never disposes it.

```
Caller creates CTS
  → passes cts.Token into async method
    → method observes token
      → throws OperationCanceledException
        → caller catches it
```

---

## How to observe cancellation

### ThrowIfCancellationRequested
Throws `OperationCanceledException` automatically. Use at checkpoints in CPU-bound loops.

```csharp
while (processing)
{
    ct.ThrowIfCancellationRequested(); // safe checkpoint
    DoWork();
}
```

### IsCancellationRequested
Manual check — lets you clean up or return a value instead of throwing.

```csharp
if (ct.IsCancellationRequested)
    return Result.Cancelled;
```

### Pass token into async API
Most common case. Pass the token into the BCL or library call and let it handle cancellation internally.

```csharp
await httpClient.GetAsync(url, ct);
await dbContext.Products.ToListAsync(ct);
await File.ReadAllTextAsync(path, ct);
```

> [!important]
> The token must flow all the way down the call chain. Accepting a token but not forwarding it into I/O calls means the operation cannot actually be cancelled.

---

## How cancellation works internally

### Case 1 — I/O (HttpClient, EF Core, file I/O)
No loop involved. The token is passed all the way down to an OS-level wait. The runtime registers a callback on the token: "if cancelled, abort the OS wait." Either the I/O completes normally or the token fires — whichever comes first. `OperationCanceledException` unwinds the stack back to your catch block.

### Case 2 — Task.Delay
`Task.Delay` creates a `System.Threading.Timer` and simultaneously calls `ct.Register(...)` with a callback that disposes the timer and marks the task cancelled. No polling — it is a race between the timer and the token callback. This is why passing `stoppingToken` into `Task.Delay` in a `BackgroundService` causes immediate exit on shutdown instead of waiting the full duration.

### Case 3 — CPU-bound code
No I/O = no OS wait = no automatic hook for the token. You must add manual checkpoints with `ThrowIfCancellationRequested()` at safe points in your loop. Reading the internal `volatile int` state flag is near-zero cost.

---

## Real-world patterns

### ASP.NET controller
ASP.NET Core automatically binds `HttpContext.RequestAborted` to a `CancellationToken` parameter — no CTS needed.

```csharp
[HttpGet]
public async Task<IActionResult> GetData(CancellationToken cancellationToken)
{
    var result = await _service.GetDataAsync(cancellationToken);
    return Ok(result);
}
```

Cancellation is triggered automatically when the client disconnects or navigates away — the browser closes the TCP connection, ASP.NET detects it and fires the token. If you want an explicit cancel button on the frontend, use JS `AbortController` — it also just closes the connection, not a separate API call.

### Linked tokens — timeout + request abort
```csharp
using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(3));
using var linked = CancellationTokenSource
    .CreateLinkedTokenSource(cancellationToken, timeoutCts.Token);

try
{
    var result = await _service.GetAsync(id, linked.Token);
    return Ok(result);
}
catch (OperationCanceledException)
{
    return cancellationToken.IsCancellationRequested
        ? StatusCode(499, "Client closed request")
        : StatusCode(504, "Gateway timeout");
}
```

### BackgroundService
```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        try
        {
            await ProcessQueueAsync(stoppingToken);
        }
        catch (OperationCanceledException) { break; }    // planned shutdown
        catch (Exception ex) { _logger.LogError(ex, "Error"); } // keep going

        await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken); // wakes immediately on shutdown
    }
}
```

---

## Common traps

> [!warning] Forgetting to dispose CancellationTokenSource
> CTS holds a `WaitHandle` (an OS kernel handle) and callback registrations. Not disposing it causes a resource leak — the OS handle stays open until the finalizer runs. Always use `using var cts = ...`.

> [!warning] Not forwarding the token through the call chain
> Accepting a token but not passing it into `GetAsync`, `SaveChangesAsync`, etc. means the operation cannot be cancelled. The token must reach the actual I/O call.

> [!warning] Swallowing OperationCanceledException
> ```csharp
> // ❌ caller cannot distinguish cancellation from other errors
> catch (OperationCanceledException) { return null; }
>
> // ✅ re-throw so the caller knows
> catch (OperationCanceledException) when (token.IsCancellationRequested) { throw; }
> ```

> [!warning] Catching TaskCanceledException instead of OperationCanceledException
> `TaskCanceledException` is a subclass. Catching only it misses `OperationCanceledException` thrown directly by `ThrowIfCancellationRequested()`. Always catch the base type.

> [!warning] Cancellation ≠ task failure
> A cancelled task has `Status == TaskStatus.Canceled`, not `Faulted`. Check `task.IsCanceled` explicitly when inspecting task status.

> [!warning] Accessing token after disposing CTS
> After `cts.Dispose()`, accessing `cts.Token` throws `ObjectDisposedException`. Cache the token before disposal if needed elsewhere.

---

## See also

- [[02-async-await-internals]]
- [[04-task-combinators]]
- [[05-synchronization-primitives]]
