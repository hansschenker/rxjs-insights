# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Directory Is

A personal knowledge repository of RxJS learning notes — plain `.txt` files compiled from AI conversations (Claude, Grok, Gemini), books (Yakov Fain), and personal research. There is no build system, no tests, and no runnable code.

**GitHub:** https://github.com/hansschenker/rxjs-insights

## File Overview

### Source notes (`.txt`)

| File | Content |
|------|---------|
| `rxjs-insights-list.txt` | Condensed list of key RxJS conceptual insights |
| `Rxjs-insights-summariy-0-evolution.txt` | Section 0: lineage from Haskell → LINQ → Rx.NET → RxJS |
| `rxjs-insights-claude-01.txt` / `rxjs-insights-claude-all.txt` | Claude-generated deep-dives |
| `rxjs-insights-grok-4.txt` / `rxjs-insights-grok-4-notebooklm.txt` | Grok 4 generated content |
| `rxjs-essential-characteristics-grok3-com.txt` | Grok 3 characteristics overview |
| `rxjs-essential-knowledge.txt` | Core RxJS knowledge reference |
| `rxjs-essentials-vs-javascript-essentials.txt` | Contrast: RxJS concepts vs vanilla JS equivalents |
| `rxjs-essentials-yakov-fain.txt` | Notes from Yakov Fain's RxJS book |
| `rxjs-essentials.txt` | Observable-as-inverted-iterator essay |
| `rxjs-fundamental-insights-chatgpt-45-version-2-refined.txt` | ChatGPT-refined fundamentals |
| `rxjs-insights-gemini.txt` | Gemini-generated insights |

### Derived analysis (`.md`)

| File | Content |
|------|---------|
| [`rxjs-insight-groups.md`](rxjs-insight-groups.md) | Index of all 14 insight groups with summaries and source references |
| [`rxjs-insight-01.md`](rxjs-insight-01.md) | Historical Lineage: Haskell → LINQ → Rx.NET → RxJS |
| [`rxjs-insight-02.md`](rxjs-insight-02.md) | Observable as `{Time, Value}` pairs; value-based vs time-based operators |
| [`rxjs-insight-03.md`](rxjs-insight-03.md) | Observer / Iterator Duality: push vs pull, inverted interfaces |
| [`rxjs-insight-04.md`](rxjs-insight-04.md) | Observable as Functor / Applicative / Monad; `flatMap` laws |
| [`rxjs-insight-05.md`](rxjs-insight-05.md) | Reactive Programming Paradigm; dependency graph; Reactive Manifesto |
| [`rxjs-insight-06.md`](rxjs-insight-06.md) | Functional Reactive Programming; pure functions, immutability, side effects at edges |
| [`rxjs-insight-07.md`](rxjs-insight-07.md) | RxJS as a DSL; operators as vocabulary; domain independence |
| [`rxjs-insight-08.md`](rxjs-insight-08.md) | Unifying Async Model; callbacks / Promises / events → one interface |
| [`rxjs-insight-09.md`](rxjs-insight-09.md) | Operator Taxonomy; three axes; family trees; 16 essential operators |
| [`rxjs-insight-10.md`](rxjs-insight-10.md) | Three-Step Workflow: Create → Pipe → Subscribe; subscription tree |
| [`rxjs-insight-11.md`](rxjs-insight-11.md) | Hot vs Cold; unicast vs multicast; `share` / `shareReplay` / Subject |
| [`rxjs-insight-12.md`](rxjs-insight-12.md) | Declarative vs Imperative; Observable does not store data |
| [`rxjs-insight-13.md`](rxjs-insight-13.md) | Practical Concerns: memory, error handling, backpressure, marble testing |
| [`rxjs-insight-14.md`](rxjs-insight-14.md) | TypeScript Integration; `Observable<T>`, `OperatorFunction<T,R>` |

## Conceptual Themes Across the Files

The themes below are derived from cross-file analysis and documented in full in [`rxjs-insight-groups.md`](rxjs-insight-groups.md).

| # | Group | Detail | One-line summary |
|---|---|---|---|
| 1 | Historical Lineage | [rxjs-insight-01.md](rxjs-insight-01.md) | Haskell → LINQ → Rx.NET → RxJS; each step added one dimension |
| 2 | `{Time, Value}` pairs | [rxjs-insight-02.md](rxjs-insight-02.md) | Observable = lazy infinite sequence of `[{T, a}…]`; operators act on T or a |
| 3 | Observer / Iterator Duality | [rxjs-insight-03.md](rxjs-insight-03.md) | Observable is an inverted Iterator: push vs pull, same algebra |
| 4 | Functor / Monad | [rxjs-insight-04.md](rxjs-insight-04.md) | Observable satisfies Functor → Applicative → Monad; `flatMap` is the monadic bind |
| 5 | Reactive Programming Paradigm | [rxjs-insight-05.md](rxjs-insight-05.md) | Change propagates through a dependency graph; Reactive Manifesto: responsive, resilient, elastic, message-driven |
| 6 | Functional Reactive Programming | [rxjs-insight-06.md](rxjs-insight-06.md) | FP principles (pure functions, immutability, composition) applied to time-varying streams |
| 7 | RxJS as a DSL | [rxjs-insight-07.md](rxjs-insight-07.md) | Operators = vocabulary; `pipe()` = grammar; domain changes, operators stay the same |
| 8 | Unifying Async Model | [rxjs-insight-08.md](rxjs-insight-08.md) | Callbacks, Promises, DOM events, EventEmitters → one `Observable` interface |
| 9 | Operator Taxonomy | [rxjs-insight-09.md](rxjs-insight-09.md) | Creation vs pipeline; first-order vs higher-order; value-based vs time-based; complex built from simple |
| 10 | Three-Step Workflow | [rxjs-insight-10.md](rxjs-insight-10.md) | Create (source) → Pipe (transform) → Subscribe (sink); nothing runs until subscribe |
| 11 | Hot vs Cold | [rxjs-insight-11.md](rxjs-insight-11.md) | Cold = unicast, starts on subscribe; Hot = multicast, runs independently; `share`/`shareReplay` bridge them |
| 12 | Declarative vs Imperative | [rxjs-insight-12.md](rxjs-insight-12.md) | Observable does not store data; values are pushed as they arrive, discarded after |
| 13 | Practical Concerns | [rxjs-insight-13.md](rxjs-insight-13.md) | Subscription cleanup, error handling (`catchError`, `retry`), backpressure, marble testing |
| 14 | TypeScript Integration | [rxjs-insight-14.md](rxjs-insight-14.md) | `Observable<T>`, `OperatorFunction<T,R>` give compile-time guarantees across `pipe()` chains |

## Working With These Files

When asked to add, expand, or synthesise content here:
- Write plain text or lightly structured Markdown — no code scaffolding
- Keep new material consistent with the conceptual framing already present (FRP, duality, monad/functor language)
- Prefer editing an existing file over creating a new one unless the topic is clearly distinct
- Name new files in lowercase kebab-case with the `rxjs-` prefix
