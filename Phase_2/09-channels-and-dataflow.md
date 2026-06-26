---
tags: [phase2, threading, async, channels, dataflow, csharp]
created: 2026-06-26
phase: 2
topic: 9
---

# T9 — Channels & Dataflow

[[#Why Channels Exist]] | [[#Bounded Channels & Backpressure]] | [[#Channel API — Wait Try and Combined Methods]] | [[#Multiple Producers & Consumers]] | [[#Why Dataflow Exists]] | [[#Core Block Types]] | [[#Wiring Blocks Together]] | [[#Concurrency & Backpressure in Dataflow]] | [[#Posting Items — Post vs SendAsync]] | [[#Completion Propagation]] | [[#Channel vs TPL Dataflow]] | [[#Common Bugs]] | [[#Mental Checklist]]

---

## Why Channels Exist

`BlockingCollection<T>` (T6) solves producer/consumer coordination, but it's **synchronous** — a reader or writer ties up a real thread while it waits. `Channel<T>` (`System.Threading.Channels`) is the async-native version of the same idea: `WriteAsync`/`ReadAsync` suspend a `Task` instead of blocking a thread.

Everything below builds on one core picture: a buffer between writers and readers, with rules for what happens once it fills up.

---

## Bounded Channels & Backpressure

A bounded channel has a fixed capacity. What happens on the next write once it's full depends on `BoundedChannelFullMode`.

| Mode | When full | Typical use |
|---|---|---|
| `Wait` (default) | `WriteAsync` suspends until a reader frees a slot | Backpressure |
| `DropOldest` | Removes the **oldest queued** item, then writes the new one | Live telemetry — only latest values matter |
| `DropNewest` | Removes the **newest queued** item, then writes the new one | Rare — keeps the first item, drops recent backlog |
| `DropWrite` | Discards the **incoming write** itself; buffer untouched | Sampling/logging where new data can be skipped |

> [!WARNING]
> `DropOldest` and `DropNewest` both *keep* the incoming item — neither one drops the new write. Only `DropWrite` discards what was just written. It's easy to assume "drop oldest" means "keep the earliest items," but the buffer actually ends up holding the most *recent* arrivals.

`TryWrite`/`TryRead` are the synchronous, non-suspending counterparts of `WriteAsync`/`ReadAsync` — they return `false` instead of awaiting.

> [!WARNING]
> Using `TryWrite` as a drop-in replacement for `WriteAsync` without checking the returned `bool` silently loses data once the buffer is full — `false` doesn't mean "wait," it means "didn't happen, and nobody told you."

```csharp
var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(5)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleReader = true,
    SingleWriter = false
});
```

Unbounded channels (`Channel.CreateUnbounded<T>()`) skip all of this — `WriteAsync` never suspends, and the failure mode shifts from "blocked writer" to "unbounded memory growth" if the consumer falls behind.

---

## Channel API — Wait, Try, and Combined Methods

A consistent three-way pattern, on both the writer and reader side:

| Method | Sync/async | What it does | Return value meaning |
|---|---|---|---|
| `TryWrite(item)` | Sync | Attempts a write right now | `true` = written; `false` = couldn't right now |
| `WaitToWriteAsync()` | Async | Suspends until a write is *likely* to succeed | `true` = go ahead and try; `false` = writer side closed, never will succeed |
| `WriteAsync(item)` | Async | Combines the two above | Throws if the channel is already completed |
| `TryRead(out item)` | Sync | Attempts a read right now | `true` = got an item; `false` = nothing buffered this instant |
| `WaitToReadAsync()` | Async | Suspends until an item is available, or it's certain none ever will be | `true` = try now; `false` = completed **and** fully drained, forever |
| `ReadAsync()` | Async | Combines the two above | Throws `ChannelClosedException` if completed before an item arrives |

```csharp
await channel.Writer.WriteAsync(42);
channel.Writer.Complete();              // signals "no more items"

await foreach (var item in channel.Reader.ReadAllAsync())
    Console.WriteLine(item);            // exits cleanly when Complete() fires
```

- `Complete()` throws if called on an already-completed channel; `TryComplete()` returns `false` instead.
- `WaitToReadAsync()` does **not** take a duration — it suspends until data arrives or the channel is fully drained. `false` specifically means "nothing will ever arrive again," not "try later."

---

## Multiple Producers & Consumers

The safe pattern for several consumers sharing one channel is the Wait+Try double loop — never `ReadAllAsync()` alone when there's more than one reader:

```csharp
async Task ConsumeAsync(ChannelReader<int> reader, string name)
{
    while (await reader.WaitToReadAsync())       // "is there something, ever again?"
    {
        while (reader.TryRead(out var value))    // actually claim items, one at a time
            Console.WriteLine($"{name}: {value}");
    }
}
```

If two consumers both see `WaitToReadAsync()` return `true`, only **one** of them succeeds on any given `TryRead` call — the other gets `false` and loops back to waiting. No item is ever handed to two consumers at once, and nothing is lost.

> [!WARNING]
> `SingleReader`/`SingleWriter` on `ChannelOptions` are performance promises, not enforced rules. Setting `SingleReader = true` and then running two consumer tasks anyway doesn't throw — it produces undefined behavior. In practice this showed up as **starvation** (one consumer got everything, the other sat suspended forever), not duplication — the lock-free fast path the runtime takes for "single reader" gets confused about who to wake up.

---

## Why Dataflow Exists

A `Channel<T>` is one pipe — you write the forwarding loop yourself. `System.Threading.Tasks.Dataflow` is a library of pre-built "blocks" you link together into a pipeline; each block already knows how to buffer, run a delegate, propagate completion, and limit its own concurrency.

---

## Core Block Types

| Block | Role | What it does |
|---|---|---|
| `BufferBlock<T>` | Buffer | Holds posted messages in a FIFO queue — no transformation |
| `TransformBlock<TIn,TOut>` | Transform | Applies a function to each input, order-preserving by default |
| `TransformManyBlock<TIn,TOut>` | Transform | Like TransformBlock, but each input can produce zero, one, or many outputs |
| `ActionBlock<T>` | Terminal | Runs a delegate per item, produces no output — usually the last stage |
| `BroadcastBlock<T>` | Fan-out | Forwards every message to **all** linked targets; overwrites, doesn't queue |
| `BatchBlock<T>` | Fan-out | Groups N items into an array before forwarding |

---

## Wiring Blocks Together

```csharp
var buffer = new BufferBlock<int>();
var transform = new TransformBlock<int, string>(x => x.ToString());
var action = new ActionBlock<string>(s => Console.WriteLine(s));

var linkOptions = new DataflowLinkOptions { PropagateCompletion = true };
buffer.LinkTo(transform, linkOptions);
transform.LinkTo(action, linkOptions);
```

Predicate links route messages conditionally:

```csharp
var even = new BufferBlock<int>();
var odd  = new BufferBlock<int>();

source.LinkTo(even, n => n % 2 == 0);  // only even numbers go here
source.LinkTo(odd);                     // catch-all — link this one last
```

> [!WARNING]
> Link order controls **message routing priority**, not just completion. A block offers each message to its linked targets in the order the links were created; the first target whose predicate accepts it gets it. A no-predicate ("catch-all") link accepts everything, so linking it *before* a predicate link starves that predicate link completely — it never receives anything, even matching items, because the catch-all already claimed the message first.

---

## Concurrency & Backpressure in Dataflow

```csharp
var resize = new TransformBlock<Image, Image>(
    img => Resize(img),
    new ExecutionDataflowBlockOptions
    {
        MaxDegreeOfParallelism = 4,   // up to 4 images resized concurrently
        BoundedCapacity = 20          // backpressure once 20 are queued
    });
```

`MaxDegreeOfParallelism` is a **sliding window**, not a batch gate — up to N items can be in flight at any moment; the instant one finishes, its slot frees immediately and the next queued item starts right then, independent of the other in-flight items.

| `EnsureOrdered` | Default | When a finished item moves to the next stage |
|---|---|---|
| `true` | ✅ | Only once every item that arrived *before* it has already moved on — a fast item can sit waiting behind a slower, earlier-arrived one |
| `false` | | Immediately, the instant it finishes — a later item can overtake an earlier one still processing |

---

## Posting Items — Post vs SendAsync

Maps directly onto `TryWrite`/`WriteAsync` from channels:

| Method | Sync/async | When `BoundedCapacity` is full |
|---|---|---|
| `Post(item)` | Sync | Returns `false` immediately — item is lost if not handled |
| `SendAsync(item)` | Async | Suspends until there's room, like `WriteAsync` |

---

## Completion Propagation

```csharp
buffer.Post(1);
buffer.Complete();
await action.Completion;
```

- `PropagateCompletion = true` on a link cascades `Complete()` downstream automatically — complete the producer and it ripples through the whole chain once each stage finishes draining.
- Without it, `Complete()` only marks that one block as done; every other block in the chain has to be completed manually, in the right order.
- `await block.Completion` waits for a specific block to be fully done — the Dataflow equivalent of awaiting a channel reader's `ReadAllAsync()` loop finishing.

---

## Channel vs TPL Dataflow

| | `Channel<T>` | TPL Dataflow |
|---|---|---|
| Unit | One pipe — you write the loop | A graph of blocks, each with its own queue |
| Composability | Manual forwarding | Built-in via `LinkTo` |
| Concurrency | None built-in — spawn tasks yourself | Per-block `MaxDegreeOfParallelism` |
| Backpressure | `BoundedChannelFullMode` | `BoundedCapacity` (+ Post/SendAsync) |
| Raw perf (single stage) | ~5.7ms / 100k jobs (2019 benchmark) | ~6.9ms / 100k jobs — close, not dramatically slower |
| Reach for it when | One producer/consumer hookup, or a single stage even with its own concurrency need | Multiple stages need to be **composed** together |

> [!NOTE]
> The decision isn't really "count the stages" — it's "does this need composition." A lone `ActionBlock<T>` for one stage needing `MaxDegreeOfParallelism` is reasonable. The real signal for Dataflow is wiring multiple stages together with completion cascading — for one pipe and nothing to compose, a channel has less API surface for roughly the same performance.

---

## Common Bugs

- **`TryWrite` ignoring the `false` return** — items silently vanish once the buffer is full; no exception, no warning.
- **Lying about `SingleReader`/`SingleWriter`** — multiple actual readers with `SingleReader = true` causes starvation, not a clean error.
- **Missing `PropagateCompletion = true`** — the single most common way a Dataflow pipeline hangs forever on `await block.Completion`.
- **Catch-all link created before a predicate link** — the predicate block silently never receives anything.
- **Assuming `DropOldest`/`DropNewest` drop the new item** — they don't; only `DropWrite` does.

---

## Mental Checklist

- [ ] Single pipe, no composition needed? → `Channel<T>`
- [ ] Multiple stages, and/or per-stage concurrency tuning? → TPL Dataflow
- [ ] Using `TryWrite`/`Post`? → am I checking the returned `bool`?
- [ ] Multiple consumers on one channel? → Wait+Try double loop, never `ReadAllAsync()` alone
- [ ] Linking a catch-all alongside predicate links? → catch-all goes **last**
- [ ] Chaining Dataflow blocks? → `PropagateCompletion = true` on every link
- [ ] Need strict per-item ordering downstream? → leave `EnsureOrdered` at its default (`true`)

---

## See also

[[08-valuetask]]
[[06-concurrent-collections]]
