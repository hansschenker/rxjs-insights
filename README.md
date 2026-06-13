# RxJS Insights

A personal knowledge repository of RxJS conceptual insights, compiled from AI conversations (Claude, Grok, Gemini), books (Yakov Fain), and personal research.

The raw notes in the `.txt` files have been cross-analysed and organised into **14 thematic insight groups**, each with a dedicated deep-dive document.

---

## Insight Groups

| # | Group | Summary |
|---|---|---|
| 1 | [Historical Lineage](rxjs-insight-01.md) | Haskell → LINQ → Rx.NET → RxJS; each step added one dimension |
| 2 | [{Time, Value} Pairs](rxjs-insight-02.md) | Observable = lazy infinite sequence of `[{T, a}…]`; operators act on T or a |
| 3 | [Observer / Iterator Duality](rxjs-insight-03.md) | Observable is an inverted Iterator: push vs pull, same algebra |
| 4 | [Functor / Applicative / Monad](rxjs-insight-04.md) | Observable satisfies Functor → Applicative → Monad; `flatMap` is the monadic bind |
| 5 | [Reactive Programming Paradigm](rxjs-insight-05.md) | Change propagates through a dependency graph; Reactive Manifesto |
| 6 | [Functional Reactive Programming](rxjs-insight-06.md) | FP principles applied to time-varying streams |
| 7 | [RxJS as a DSL](rxjs-insight-07.md) | Operators = vocabulary; `pipe()` = grammar; domain changes, operators stay the same |
| 8 | [Unifying Async Model](rxjs-insight-08.md) | Callbacks, Promises, DOM events, EventEmitters → one `Observable` interface |
| 9 | [Operator Taxonomy](rxjs-insight-09.md) | Creation vs pipeline; first-order vs higher-order; value-based vs time-based |
| 10 | [Three-Step Workflow](rxjs-insight-10.md) | Create (source) → Pipe (transform) → Subscribe (sink) |
| 11 | [Hot vs Cold](rxjs-insight-11.md) | Cold = unicast; Hot = multicast; `share` / `shareReplay` bridge them |
| 12 | [Declarative vs Imperative](rxjs-insight-12.md) | Observable does not store data; values are pushed as they arrive |
| 13 | [Practical Concerns](rxjs-insight-13.md) | Memory, error handling, backpressure, marble testing |
| 14 | [TypeScript Integration](rxjs-insight-14.md) | `Observable<T>`, `OperatorFunction<T,R>` give compile-time guarantees |

The full index with source references for each group is in [rxjs-insight-groups.md](rxjs-insight-groups.md).

---

## Source Notes

The `.txt` files are the raw material these analyses were derived from:

| File | Content |
|------|---------|
| `rxjs-insights-list.txt` | Condensed list of key RxJS conceptual insights |
| `Rxjs-insights-summariy-0-evolution.txt` | Lineage from Haskell → LINQ → Rx.NET → RxJS |
| `rxjs-insights-claude-01.txt` / `rxjs-insights-claude-all.txt` | Claude-generated deep-dives |
| `rxjs-insights-grok-4.txt` / `rxjs-insights-grok-4-notebooklm.txt` | Grok 4 generated content |
| `rxjs-essential-characteristics-grok3-com.txt` | Grok 3 characteristics overview |
| `rxjs-essential-knowledge.txt` | Core RxJS knowledge reference |
| `rxjs-essentials-vs-javascript-essentials.txt` | RxJS concepts vs vanilla JS equivalents |
| `rxjs-essentials-yakov-fain.txt` | Notes from Yakov Fain's RxJS book |
| `rxjs-essentials.txt` | Observable-as-inverted-iterator essay |
| `rxjs-fundamental-insights-chatgpt-45-version-2-refined.txt` | ChatGPT-refined fundamentals |
| `rxjs-insights-gemini.txt` | Gemini-generated insights |
