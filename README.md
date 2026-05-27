# 📚 DotNet Learning Notes

Personal study notes for C#/.NET interview preparation.

## Structure

Notes are organized by phase, following a structured 9-phase learning plan ordered by interview priority.

| Phase | Topic | Status | Notes |
|-------|-------|--------|-------|
| Phase 2 | Threading & Async | 🔄 In progress | 4/8 topics |
| Phase 3 | C# Internals | ⏳ Upcoming | |
| Phase 5 | Design Patterns | ⏳ Upcoming | |
| Phase 4 | ASP.NET Core & EF Core | ⏳ Upcoming | |
| Phase 1 | Memory & GC | ✅ Complete | 12 notes |
| Phase 7 | Testing | ⏳ Upcoming | |
| Phase 8 | Web / Auth / Security | ⏳ Upcoming | |
| Phase 9 | System Design | ⏳ Upcoming | |
| Phase 6 | Algorithms | ⏳ Upcoming | |

---

## Phase 1 — Memory & GC ✅

| # | Note | Topic |
|---|------|-------|
| 01 | [weak-reference](Phase_1/01-weak-reference.md) | WeakReference\<T\> |
| 02 | [span-memory](Phase_1/02-span-memory.md) | Span\<T\> / Memory\<T\> |
| 03 | [arraypool](Phase_1/03-arraypool.md) | ArrayPool\<T\> |
| 04 | [equals-gethashcode](Phase_1/04-equals-gethashcode.md) | Equals / GetHashCode contract |
| 05 | [iequatable](Phase_1/05-iequatable.md) | IEquatable\<T\> |
| 06 | [record-equality](Phase_1/06-record-equality.md) | Record equality |
| 07 | [hash-collections](Phase_1/07-hash-collections.md) | Hash collections internals |
| 08 | [hashcode-combine](Phase_1/08-hashcode-combine.md) | HashCode.Combine |
| 09 | [memory-gc-internals](Phase_1/09-memory-gc-internals.md) | GC generations / LOH |
| 10 | [class-struct-record](Phase_1/10-class-struct-record.md) | Class vs struct vs record |
| 11 | [compile-time-vs-runtime](Phase_1/11-compile-time-vs-runtime.md) | Compile-time vs runtime |
| 12 | [struct-parameter-passing](Phase_1/12-struct-parameter-passing.md) | Struct parameter passing |

---

## Phase 2 — Threading & Async 🔄

| # | Note | Topic | Status |
|---|------|-------|--------|
| 01 | [task-thread-threadpool](Phase_2/01-task-thread-threadpool.md) | Task vs Thread vs ThreadPool | ✅ |
| 02 | [async-await-internals](Phase_2/02-async-await-internals.md) | async/await state machine | ✅ |
| 03 | [cancellation-token](Phase_2/03-cancellation-token.md) | CancellationToken | ✅ |
| 04 | [task-combinators](Phase_2/04-task-combinators.md) | WhenAll / WhenAny / WhenEach | ✅ |
| 05 | — | Synchronization primitives | ⏳ |
| 06 | — | Deadlocks & race conditions | ⏳ |
| 07 | — | ValueTask | ⏳ |
| 08 | — | Channels / Dataflow | ⏳ |

---

## Tools

Notes follow Obsidian formatting conventions (YAML frontmatter, callouts, wiki links).
