---
phase: 3
topic: Delegates & Events
status: complete
tags: [csharp, delegates, events, multicast, memory-leaks]
---

# Delegates & Events

## Core Idea

A **delegate** is a type-safe object that holds a reference to a method — you can pass it around like a variable: store it, pass it as an argument, return it, invoke it later.

An **event** is a thin wrapper around a delegate field that restricts what external code can do with it. It's not a separate mechanism — `event` is a C# keyword that changes the *accessibility* of a delegate field. All the underlying machinery is still delegates.

## What a Delegate Actually Is

```csharp
public delegate int MathOp(int a, int b);

int Add(int a, int b) => a + b;

MathOp op = Add;
int result = op(3, 4); // 7
```

The compiler generates a full class behind `MathOp`, deriving from `System.MulticastDelegate` → `System.Delegate`. That generated object holds:

- `_target` — instance reference (null if the method is static)
- `_methodPtr` — pointer to the actual method
- `_invocationList` — array, used only when multicast

This is why delegates are "type-safe function pointers" — the type itself encodes the exact signature, checked at compile time, not after a crash like a raw C function pointer.

> [!NOTE]
> Rarely declare custom delegate types anymore — `Action`, `Action<T>`, `Func<T,TResult>`, and `Predicate<T>` cover almost everything.

## Multicast — One Delegate, Many Methods

```csharp
Action greet = () => Console.WriteLine("Hi");
greet += () => Console.WriteLine("Hello");
greet += () => Console.WriteLine("Hey");

greet(); // Hi, Hello, Hey — in order
```

**Delegates are immutable.** `+=` doesn't mutate the existing object — it calls `Delegate.Combine`, which builds a *new* `MulticastDelegate` with a merged invocation list. The variable is just reassigned to point at the new object.

### The Func/return-value trap

```csharp
Func<int> f = () => 1;
f += () => 2;
f += () => 3;

int result = f(); // 3, not 6
```

All three handlers run, but invoking a multicast delegate just walks the invocation list calling each method in order, overwriting a single result variable each time — only the last one survives. There's no mechanism to collect multiple results because the signature only has room for one return value per call:

```
[lambda1] → [lambda2] → [lambda3]
 returns 1   returns 2   returns 3
 discarded   discarded   kept, returned
```

This is exactly why `void`-returning delegates (`Action`, `EventHandler`) are the natural fit for multicast — nothing gets silently lost. Multicasting a `Func<T>` is a footgun: it compiles, all handlers run (side effects included), but only one result comes back.

To get every result, walk the list manually:

```csharp
foreach (Func<int> d in f.GetInvocationList())
{
    Console.WriteLine(d()); // 1, 2, 3 individually
}
```

### Removing from the chain

```csharp
greet -= handler; // Delegate.Remove — new object again, minus that method
```

Equality for removal is checked per invocation-list entry (target + method) — you can't remove a lambda you never kept a reference to, because you can't identify it.

## Events — A Restricted Delegate

```csharp
public class Button
{
    public event EventHandler Clicked;

    public void SimulateClick()
    {
        Clicked?.Invoke(this, EventArgs.Empty);
    }
}
```

| | Plain delegate field | `event` field |
|---|---|---|
| External code: `btn.Clicked = null;` | ✅ allowed (wipes everyone) | ❌ compile error |
| External code: `btn.Clicked(...)` | ✅ allowed | ❌ compile error |
| External code: `btn.Clicked += h;` | ✅ allowed | ✅ allowed (the point of events) |
| Inside the declaring class | full access | full access, looks like a plain field |

**Why this design exists:** without `event`, any external code holding the delegate field could invoke it directly or null out everyone else's subscriptions. `event` restricts outsiders to `+=`/`-=` only — publishing stays exclusive to the declaring class. It only ever *takes away* capabilities compared to a plain field; it never grants new ones.

### The null-check pattern

```csharp
Clicked?.Invoke(this, EventArgs.Empty);
```

If nobody's subscribed, the field is `null` — invoking it directly throws `NullReferenceException`, hence `?.`.

**Thread-safety nuance:** even with `?.`, a race exists — between the null-check and the invoke, another thread could remove the last handler. Idiomatic fix: capture into a local first.

```csharp
var handler = Clicked;
handler?.Invoke(this, EventArgs.Empty);
```

This works because delegates are immutable — a concurrent `-=` elsewhere creates a *new* object and doesn't affect your local copy.

### Custom add/remove

```csharp
private EventHandler _clicked;
public event EventHandler Clicked
{
    add { _clicked += value; }
    remove { _clicked -= value; }
}
```

The simple `public event EventHandler Clicked;` syntax is sugar for exactly this — compiler-generated backing field plus add/remove accessors. Write the explicit version when you need custom logic (thread-safe add/remove via `Interlocked`, forwarding to another object's event, etc.).

## Subscribing — The Concept

An event is a notification board: the publisher posts "something happened," subscribers hand over a method to run when it does. The publisher doesn't know or care what the subscriber does with the notification.

**Example — a `Timer` driving a `Form`'s clock display:**

```csharp
public class Timer
{
    public event EventHandler Tick;

    public void Start()
    {
        // every 1 second:
        Tick?.Invoke(this, EventArgs.Empty);
    }
}

public class Form
{
    private Label clockLabel;

    public Form(Timer timer)
    {
        timer.Tick += OnTick; // "when you tick, call my method"
    }

    void OnTick(object sender, EventArgs e)
    {
        clockLabel.Text = DateTime.Now.ToString("HH:mm:ss");
    }
}
```

Direction of control: the **Timer drives timing**, the **Form reacts**. The Form subscribes because it has no way of knowing *when* a second has passed on its own.

### The alternative: polling

Without events, the Form would have to keep asking "has it happened yet?" in a loop — wasteful, and someone has to own that loop.

| | Polling | Events |
|---|---|---|
| Who checks | Subscriber repeatedly asks | Publisher notifies once, exactly when it happens |
| Wasted work | Many checks per real event | Zero — notified exactly once per occurrence |
| Multiple subscribers | Each needs its own redundant loop | Each just adds itself to the invocation list once |
| Control direction | Subscriber **pulls** | Publisher **pushes** |

Events exist to flip pull into push — the thing that knows when something happens is responsible for telling everyone, instead of everyone wastefully asking.

## Memory Leak Gotcha

Subscribing a **short-lived** object's handler to a **long-lived** publisher's event keeps that object alive — the delegate's `_target` field is a strong reference.

**Scenario:** a long-lived `Timer` (lives for the whole app), a short-lived `Form` (opened/closed repeatedly by the user).

```
Timer (lives forever)
 └─ Tick (event field)
     └─ invocation list: [delegate1, delegate2, ...]
                              │ _target
                              ▼
                         Form instance #1  ← still alive even though "closed"!
                         Form instance #2  ← still alive!
```

1. `timer.Tick += OnTick` stores a delegate whose `_target` points at this specific `Form` instance, inside `Timer`'s invocation list.
2. User closes the Form. The Timer has no idea — it still holds that delegate.
3. GC checks "is anything pointing at this Form?" — yes, `Timer.Tick` still is. Can't collect it.
4. Repeat across many open/close cycles → many "zombie" Forms pinned in memory.

**Fix:** always `-=` on disposal when the subscriber's lifetime is shorter than the publisher's:

```csharp
timer.Tick -= OnTick;
```

This removes the entry from the invocation list, breaking the reference chain so the GC can collect the Form.

## Self-Check

- [x] What fields live inside the compiler-generated delegate class, and what each is for
- [x] Why `+=` on a delegate creates a new object instead of mutating the old one
- [x] Why a multicast `Func` only returns the last handler's result
- [x] Exactly what `event` restricts that a plain delegate field would allow
- [x] Why capturing a local copy before invoking is safer under concurrency
- [x] Why subscribing to a long-lived publisher's event can leak memory

## See Also

- [[Phase_3]]
- [[Closures]] (later in Phase 3 — closures are how lambdas capture `this` and locals, directly relevant to the `_target` reference discussed above)
