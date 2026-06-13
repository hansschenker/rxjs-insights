# Insight 09 — Operator Taxonomy

> Part of [rxjs-insight-groups.md](rxjs-insight-groups.md)  
> Primary sources: [rxjs-essential-knowledge.txt](rxjs-essential-knowledge.txt), [rxjs-insights-grok-4.txt](rxjs-insights-grok-4.txt), [rxjs-insights-gemini.txt](rxjs-insights-gemini.txt), [rxjs-essentials-yakov-fain.txt](rxjs-essentials-yakov-fain.txt)

---

## Why a Taxonomy?

RxJS ships with over 100 operators. Trying to learn them as a flat list is overwhelming and ineffective. A taxonomy — a structured classification — reveals the **underlying patterns** that unify the operators and makes the library learnable in layers.

Three orthogonal axes classify every operator:

1. **Role** — creation vs pipeline
2. **Order** — first-order vs higher-order
3. **Aspect** — value-based vs time-based

---

## Axis 1 — Role: Creation vs Pipeline

### Creation Operators

Creation operators are **factory functions** that produce a new Observable from scratch. They are the entry point into the RxJS world — the only way to create a source stream:

| Operator | Produces |
|---|---|
| `of(1, 2, 3)` | Observable that emits 1, 2, 3 then completes |
| `from([1, 2, 3])` | Same, from an array or iterable |
| `from(promise)` | Observable that emits the promise's resolved value |
| `fromEvent(el, 'click')` | Observable of DOM events |
| `interval(ms)` | Observable emitting 0, 1, 2, … every `ms` milliseconds |
| `timer(delay, ms)` | Like `interval` but with an initial delay |
| `range(start, count)` | Observable emitting a range of integers |
| `ajax(url)` | Observable of an HTTP response |
| `EMPTY` | Observable that completes immediately with no emissions |
| `NEVER` | Observable that never emits and never completes |
| `throwError(fn)` | Observable that immediately errors |

### Pipeline (Pipeable) Operators

Pipeline operators are used inside `pipe()`. They take an Observable and return a new Observable — they do not create sources, they **transform** existing ones:

```typescript
source$.pipe(
  map(x => x * 2),        // pipeline operator
  filter(x => x > 5),     // pipeline operator
  take(10)                 // pipeline operator
);
```

Every pipeline operator has the same signature: `(source: Observable<T>) => Observable<R>`.

---

## Axis 2 — Order: First-Order vs Higher-Order

### First-Order Operators

A **first-order** operator maps a value of type `T` to a value of type `R`. No Observables are involved in the mapping — the operator works on plain values:

```typescript
source$.pipe(
  map((x: number) => x * 2),     // number → number
  filter((x: number) => x > 5),  // keeps or drops numbers
  scan((acc, x) => acc + x, 0)   // accumulates numbers
)
```

### Higher-Order Operators

A **higher-order** operator maps a value of type `T` to an `Observable<R>`, creating an Observable of Observables. It then **flattens** that inner Observable into the outer stream using one of four strategies:

| Operator | Strategy | Analogy |
|---|---|---|
| `mergeMap(fn)` | Subscribe to all inner Observables in parallel | All lanes open on the motorway |
| `concatMap(fn)` | Queue inner Observables; start next only when current completes | Single checkout lane, FIFO |
| `switchMap(fn)` | Cancel the current inner Observable when a new value arrives | Only the latest matters |
| `exhaustMap(fn)` | Ignore new values while an inner Observable is active | "Come back when I'm done" |

The choice of flattening operator is often the most important decision in a pipeline:

```typescript
// Search typeahead: cancel previous request on every keystroke
searchInput$.pipe(
  switchMap(q => ajax.getJSON(`/search?q=${q}`))
);

// Sequential file upload: process one at a time, in order
fileList$.pipe(
  concatMap(file => upload(file))
);

// Independent analytics events: all fire in parallel
events$.pipe(
  mergeMap(event => logToServer(event))
);

// Login button: ignore clicks while login is in progress
loginClick$.pipe(
  exhaustMap(() => authService.login(credentials))
);
```

---

## Axis 3 — Aspect: Value-Based vs Time-Based

### Value-Based Operators

Act on the **value** dimension (`a`) of the `{T, a}` model. The timing of emissions is irrelevant to these operators:

**Transform:**
| Operator | Action |
|---|---|
| `map(fn)` | Apply `fn` to each value |
| `scan(fn, seed)` | Running accumulation — emits each intermediate result |
| `reduce(fn, seed)` | Full accumulation — emits only the final result |
| `pairwise()` | Emit consecutive pairs `[prev, curr]` |
| `pluck(key)` | Extract a property (deprecated; use `map`) |

**Filter:**
| Operator | Action |
|---|---|
| `filter(pred)` | Pass only values satisfying `pred` |
| `take(n)` | Pass only the first `n` values |
| `skip(n)` | Drop the first `n` values |
| `takeWhile(pred)` | Pass values while `pred` is true, then complete |
| `skipWhile(pred)` | Drop values while `pred` is true, then pass all |
| `distinct()` | Pass only values not seen before |
| `distinctUntilChanged()` | Pass a value only if it differs from the previous one |
| `first(pred?)` | Pass only the first (matching) value, then complete |
| `last(pred?)` | Pass only the last (matching) value on completion |

**Combine (value-driven):**
| Operator | Action |
|---|---|
| `combineLatest([…])` | Emit latest value from each source whenever any emits |
| `zip([…])` | Emit paired values — waits for all sources to emit |
| `withLatestFrom(other$)` | Sample `other$` at each emission of the source |
| `forkJoin([…])` | Wait for all to complete, emit their last values together |

### Time-Based Operators

Act on the **time** dimension (`T`). The values themselves may be unchanged; it is the *when* that is transformed:

**Rate control:**
| Operator | Action |
|---|---|
| `debounceTime(ms)` | Emit only after `ms` of silence |
| `throttleTime(ms)` | Emit the first value in each `ms` window |
| `auditTime(ms)` | Emit the last value after each `ms` window |
| `sampleTime(ms)` | Emit the latest value every `ms` |

**Time shift:**
| Operator | Action |
|---|---|
| `delay(ms)` | Shift all emissions forward by `ms` |
| `delayWhen(fn)` | Delay each emission by a duration determined by `fn` |

**Time boundary:**
| Operator | Action |
|---|---|
| `timeout(ms)` | Error if no emission arrives within `ms` |
| `timeoutWith(ms, other$)` | Switch to `other$` on timeout |

**Batch by time:**
| Operator | Action |
|---|---|
| `bufferTime(ms)` | Collect emissions into arrays over `ms` windows |
| `windowTime(ms)` | Emit a nested Observable for each `ms` window |

**Creation (time-based):**
| Operator | Action |
|---|---|
| `interval(ms)` | Emit 0, 1, 2, … every `ms` |
| `timer(delay, ms?)` | Emit once after `delay`, then optionally like `interval` |

---

## Complex Operators Built from Simpler Ones

The most important structural insight of RxJS operator design: **complex operators are composed from primitive ones**. The family trees follow a consistent pattern:

```
BASE OPERATOR
  └─ BASE + All         (apply to all inner Observables)
       └─ BASE + Map    (map values to inner Observables, then apply base strategy)
            └─ BASE + MapTo  (same, but with a constant instead of a function)
```

| Base | All | Map | MapTo |
|---|---|---|---|
| `merge` | `mergeAll` | `mergeMap` | `mergeMapTo` |
| `switch` | `switchAll` | `switchMap` | `switchMapTo` |
| `concat` | `concatAll` | `concatMap` | `concatMapTo` |
| `exhaust` | `exhaustAll` | `exhaustMap` | `exhaustMapTo` |

Additional families:

```
combine  → combineAll → combineLatest → combineLatestWith
                                      → combineLatestAll
zip      → zipAll
         → zipWith
scan     → switchScan
         → mergeScan
```

Conditional families (`when` suffix):
```
window  → windowWhen, windowCount, windowToggle, windowTime
buffer  → bufferWhen, bufferCount, bufferToggle, bufferTime
repeat  → repeatWhen
retry   → retryWhen
throttle → throttleTime
debounce → debounceTime
distinct → distinctUntilChanged, distinctUntilKeyChanged
```

---

## The 16 Most Essential Operators

If forced to choose a minimal set that covers the vast majority of RxJS usage, split evenly between value-based and time-based:

**8 value-based:**
1. `map` — transform every value
2. `filter` — discard unwanted values
3. `scan` — accumulate running state
4. `reduce` — collapse a finite stream to one value
5. `take` — limit the stream to `n` values
6. `mergeMap` — parallel higher-order flattening
7. `switchMap` — latest-only higher-order flattening
8. `combineLatest` — derive state from multiple sources

**8 time-based:**
1. `debounceTime` — wait for silence
2. `throttleTime` — rate-limit emissions
3. `delay` — shift emissions in time
4. `audit` / `auditTime` — emit last value in a window
5. `sample` / `sampleTime` — periodic snapshot
6. `timeout` — enforce maximum wait time
7. `interval` — regular tick source
8. `buffer` / `bufferTime` — batch emissions into arrays

These 16 can explain the overwhelming majority of RxJS programs — all others are compositions, specialisations, or variants of these primitives.
