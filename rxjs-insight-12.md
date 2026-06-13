# Insight 12 — Declarative Dataflow vs Imperative Control Flow

> Part of [rxjs-insight-groups.md](rxjs-insight-groups.md)  
> Primary sources: [rxjs-essentials-vs-javascript-essentials.txt](rxjs-essentials-vs-javascript-essentials.txt), [rxjs-essential-characteristics-grok3-com.txt](rxjs-essential-characteristics-grok3-com.txt), [rxjs-insights-claude-01.txt](rxjs-insights-claude-01.txt), [rxjs-essentials-yakov-fain.txt](rxjs-essentials-yakov-fain.txt)

---

## The Fundamental Contrast

Two programming styles are at odds in modern JavaScript:

**Imperative control flow** — you tell the computer *how* to achieve a result, step by step. You manage state, control loops, coordinate updates, and handle timing explicitly.

**Declarative dataflow** — you tell the computer *what* you want. You define relationships between values. The system figures out when and how to propagate changes.

RxJS is firmly declarative. Understanding why requires seeing the same problem written both ways.

---

## The Canonical Example: Search with Debounce

### Imperative Version

```typescript
let lastQuery = '';
let lastResults: Result[] = [];
let pendingTimeout: number | null = null;
let pendingRequest: AbortController | null = null;

inputEl.addEventListener('input', (e) => {
  const query = (e.target as HTMLInputElement).value;

  if (query === lastQuery) return;
  lastQuery = query;

  if (pendingTimeout !== null) clearTimeout(pendingTimeout);
  if (pendingRequest !== null) pendingRequest.abort();

  if (query.length < 2) {
    renderResults([]);
    return;
  }

  pendingTimeout = window.setTimeout(async () => {
    const controller = new AbortController();
    pendingRequest = controller;

    try {
      const res = await fetch(`/search?q=${query}`, {
        signal: controller.signal
      });
      lastResults = await res.json();
      if (lastQuery === query) { // is the query still current?
        renderResults(lastResults);
      }
    } catch (err) {
      if (err.name !== 'AbortError') renderError(err);
    } finally {
      pendingTimeout = null;
      pendingRequest = null;
    }
  }, 300);
});
```

This is approximately 35 lines. The state variables (`lastQuery`, `pendingTimeout`, `pendingRequest`) must all be coordinated manually. The bug surface is large: what if `clearTimeout` is called before the timeout fires? What if two requests complete out of order? What if the component unmounts mid-request?

### Declarative Version (RxJS)

```typescript
fromEvent<InputEvent>(inputEl, 'input').pipe(
  map(e => (e.target as HTMLInputElement).value),
  filter(q => q.length >= 2),
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(q => ajax.getJSON<Result[]>(`/search?q=${q}`).pipe(
    catchError(() => of([]))
  ))
).subscribe(renderResults);
```

9 lines. No state variables. Cancellation is automatic (`switchMap` cancels the previous request). Debouncing is one operator. Deduplication is one operator. Error handling is one operator.

---

## The Contrast Table

| Dimension | Imperative | Declarative (RxJS) |
|---|---|---|
| **Expression of intent** | *How* to execute | *What* to compute |
| **State** | Mutable variables managed by hand | Immutable emissions through operators |
| **Control flow** | `for`, `while`, `if/else`, callbacks | Operators (`filter`, `takeWhile`, `switchMap`) |
| **Async coordination** | Manual: flags, timeouts, abort controllers | Structural: `switchMap`, `concatMap`, `exhaustMap` |
| **Error handling** | `try/catch`, `if (err)` scattered | `catchError`, `retry` in the stream |
| **Cancellation** | Manual: `clearTimeout`, `removeEventListener`, `AbortController` | Automatic: `unsubscribe()` tears down all |
| **Race conditions** | Must be prevented manually | Prevented structurally by operator choice |
| **Composition** | Hard — glue code required | Natural — `pipe()` |
| **Testability** | High coupling to timing and state | Pure functions, marble testing |

---

## Observable Does Not Hold Data

One of the most important conceptual shifts from imperative programming:

**An Observable does not store values.** It is not a container like an array. It is a declaration of *how values will flow* when subscribed to.

Compare:

```typescript
// Array: holds all values in memory simultaneously
const arr = [1, 2, 3, 4, 5];
// All 5 numbers exist in memory right now

// Observable: holds none of its values
const obs$ = of(1, 2, 3, 4, 5);
// No values in memory yet — they will be pushed when subscribed
```

When you subscribe:

1. The Observable pushes `1` → subscriber processes `1` → `1` is gone.
2. The Observable pushes `2` → subscriber processes `2` → `2` is gone.
3. … and so on.

Each `{T, a}` pair is delivered once and discarded. This is what makes Observables suitable for infinite streams — an array of infinite size would require infinite memory; an infinite Observable uses constant memory.

---

## Push-Based: Values Arrive, Not Requested

In the imperative model you *request* data:

```typescript
// Pull: you ask for data when you want it
const value = readSensor();         // synchronous pull
const value = await fetchData();    // async pull (but still caller-initiated)
```

In the reactive model data *arrives*:

```typescript
// Push: data arrives when it's ready; you react
sensor$.subscribe(value => processReading(value));
```

This is not merely syntactic. It reflects a fundamentally different relationship between producer and consumer:
- **Imperative**: the consumer controls timing. You call `readSensor()` when you want a reading.
- **Reactive**: the producer controls timing. `sensor$` pushes a reading when one is available.

For user interfaces and real-time data, the producer-controlled model is more natural — the user clicks when they click, not when you poll.

---

## Statements vs Expressions

Imperative code is built of **statements** — instructions that have effects:

```typescript
count++;
updateDisplay(count);
saveToDatabase(count);
```

Declarative code is built of **expressions** — computations that produce values:

```typescript
const count$ = clicks$.pipe(scan(acc => acc + 1, 0));
const doubled$ = count$.pipe(map(n => n * 2));
```

`count$` and `doubled$` are values — they can be passed to functions, stored in variables, and composed with other expressions. Statements cannot.

The declarative style makes the *data relationships* visible in the structure of the code, rather than buried in sequencing logic.

---

## The Sync / Async Parallel

The imperative / declarative contrast maps onto the synchronous / asynchronous contrast:

| | Synchronous | Asynchronous |
|---|---|---|
| **Imperative** | `const x = getValue()` | `const x = await getValueAsync()` |
| **Declarative** | `from([1,2,3]).pipe(map(fn))` | `fromEvent(source, 'data').pipe(map(fn))` |

RxJS treats synchronous and asynchronous sequences uniformly — the same operators work on both. `from([1,2,3])` and `fromEvent(button, 'click')` are both Observables; you `pipe()` the same operators over them and subscribe the same way.

This uniformity is itself a declarative achievement: you declare *what transformation you want*, and RxJS handles whether it executes synchronously or asynchronously.

---

## When Not to Use Declarative

Declarative dataflow is not universally better. It is optimised for:
- Streams with multiple values over time.
- Async coordination between multiple sources.
- Transformations that compose from named, reusable operators.

It is overkill for:
- Simple one-shot async operations that a `Promise` handles cleanly.
- Trivial synchronous transformations that plain functions express more clearly.
- Scenarios where the imperative version is genuinely simpler and shorter.

The honest position: RxJS and imperative code are not enemies. A well-designed application uses declarative streams for its complex async flows and plain functions/Promises for its simple ones.
