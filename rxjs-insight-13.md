# Insight 13 — Practical Concerns: Memory, Errors, Backpressure, Testing

> Part of [rxjs-insight-groups.md](rxjs-insight-groups.md)  
> Primary sources: [rxjs-insights-grok-4.txt](rxjs-insights-grok-4.txt), [rxjs-insights-grok-4-notebooklm.txt](rxjs-insights-grok-4-notebooklm.txt), [rxjs-essentials-yakov-fain.txt](rxjs-essentials-yakov-fain.txt), [rxjs-insights-claude-01.txt](rxjs-insights-claude-01.txt)

---

## Overview

The theoretical elegance of RxJS (see Insights 04–07) is only useful in production if you can avoid its practical pitfalls. Four areas demand attention in every real-world RxJS application:

1. **Subscription lifecycle** — memory leaks from forgotten subscriptions.
2. **Error handling** — errors that terminate streams prematurely.
3. **Backpressure** — producers emitting faster than consumers can process.
4. **Testing** — asserting the behaviour of asynchronous streams.

---

## 1. Subscription Lifecycle and Memory Management

### The Problem

Every subscription activates a chain of event listeners, timers, or HTTP connections. If you subscribe and never unsubscribe, those resources remain held indefinitely — even after the component or context that created them is gone. This is the most common source of memory leaks in RxJS applications.

```typescript
// Leak: no unsubscription
class SearchComponent {
  ngOnInit() {
    interval(1000).subscribe(() => this.poll());
    // The interval continues after this component is destroyed
  }
}
```

### The Solutions

**Explicit unsubscribe:**
```typescript
class SearchComponent {
  private sub: Subscription;

  ngOnInit() {
    this.sub = interval(1000).subscribe(() => this.poll());
  }

  ngOnDestroy() {
    this.sub.unsubscribe(); // cleans up the interval and all downstream operators
  }
}
```

**`takeUntil(destroy$)` — the idiomatic pattern for component-scoped streams:**
```typescript
class SearchComponent {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    interval(1000).pipe(
      takeUntil(this.destroy$)   // automatically completes when destroy$ emits
    ).subscribe(() => this.poll());

    fromEvent(document, 'click').pipe(
      takeUntil(this.destroy$)   // one destroy$ cleans up all streams
    ).subscribe(handler);
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**`takeWhile(predicate)` — condition-based termination:**
```typescript
poller$.pipe(
  takeWhile(() => this.isActive)
).subscribe(handler);
```

**`take(n)` — finite stream that self-completes:**
```typescript
ajax.getJSON('/api/data').pipe(
  take(1) // HTTP requests complete naturally, but take(1) makes it explicit
).subscribe(handler);
```

**Framework tools (Angular `async` pipe):**
```html
<!-- No manual subscription or unsubscription needed -->
<div *ngIf="data$ | async as data">{{ data.name }}</div>
```

### The Subscription Tree

When you unsubscribe from the leaf, the cleanup propagates upward through the entire subscription tree automatically (see Insight 10). You never need to track intermediate subscriptions.

---

## 2. Error Handling

### The Problem

An uncaught error in an Observable stream **terminates the stream**. After `observer.error(e)` is called, no more `next` values are delivered — the stream is dead. If you want the stream to survive errors, you must handle them explicitly.

```typescript
// The stream dies on first HTTP error
httpRequest$.subscribe({
  next: data => render(data),
  error: err => console.error(err) // stream is now dead; no more requests
});
```

### `catchError` — intercept and substitute

```typescript
const safe$ = httpRequest$.pipe(
  catchError(err => {
    logError(err);
    return of(defaultValue); // substitute a fallback Observable; stream continues
  })
);
```

`catchError` receives the error and must return a new Observable. The stream continues with that Observable's emissions.

### `retry(n)` — resubscribe on error

```typescript
const resilient$ = httpRequest$.pipe(
  retry(3), // resubscribe to the source up to 3 times on error
  catchError(err => of(defaultValue)) // final fallback after 3 failures
);
```

### `retryWhen(fn)` / `retry({ delay, count })` — custom retry strategy

```typescript
// Exponential backoff: wait 1s, 2s, 4s between retries
const withBackoff$ = httpRequest$.pipe(
  retry({
    count: 3,
    delay: (error, retryCount) => timer(Math.pow(2, retryCount - 1) * 1000)
  })
);
```

### Failover Pattern

```typescript
// Try primary → fallback → cache → default
const data$ = primarySource$.pipe(
  catchError(() => fallbackSource$),
  catchError(() => cachedSource$),
  catchError(() => of(hardcodedDefault))
);
```

### The Error Contract

An Observable can emit:
- Many `next(value)` — zero or more values.
- At most one `error(err)` — terminates the stream.
- At most one `complete()` — terminates the stream.

After `error` or `complete`, the stream is closed. `catchError` works by subscribing to a *new* Observable that replaces the errored one — it does not "resume" the original.

---

## 3. Backpressure

### The Problem

**Backpressure** occurs when a producer emits values faster than the consumer can process them. Left unhandled this leads to:
- Memory growth (buffered values waiting to be processed).
- Degraded responsiveness.
- Dropped or stale values reaching the consumer.

```typescript
// Problem: mouse moves fire at 60+ fps; processing might take 100ms each
fromEvent(document, 'mousemove').pipe(
  mergeMap(event => expensiveProcessing(event)) // queue grows unboundedly
)
```

### Rate-Limiting Strategies

**`debounceTime(ms)` — emit only after silence:**  
Best for inputs where only the final state matters (search field, resize handle).
```typescript
fromEvent(input, 'input').pipe(debounceTime(300))
// Only emits after the user stops typing for 300ms
```

**`throttleTime(ms)` — emit the first, suppress the rest:**  
Best for rate-limiting actions (mouse move processing, scroll handlers).
```typescript
fromEvent(document, 'mousemove').pipe(throttleTime(16))
// Max ~60 emissions per second regardless of source rate
```

**`auditTime(ms)` — emit the last in each window:**  
Best for capturing the most recent state at a fixed rate.
```typescript
fromEvent(document, 'mousemove').pipe(auditTime(16))
// Emits the latest position at most every 16ms
```

**`sampleTime(ms)` — periodic snapshot:**  
Emits the most recent value every `ms`, regardless of whether a new value arrived.
```typescript
sensor$.pipe(sampleTime(100)) // snapshot every 100ms
```

### Batching Strategies

**`bufferTime(ms)` — collect into arrays:**  
```typescript
highFreqSource$.pipe(
  bufferTime(100),           // collect 100ms of emissions into an array
  filter(batch => batch.length > 0),
  mergeMap(batch => processBatch(batch)) // process in bulk
)
```

**`bufferCount(n)` — collect into fixed-size batches:**  
```typescript
events$.pipe(
  bufferCount(50),
  mergeMap(batch => bulkInsert(batch))
)
```

### Concurrency Limiting

For higher-order operators, limit the number of concurrent inner subscriptions:

```typescript
requests$.pipe(
  mergeMap(req => httpRequest(req), 3) // max 3 simultaneous requests
)
```

---

## 4. Testing with Marble Notation

### The Problem

Testing async code is hard. Real timers make tests slow and flaky. Promises resolve at unpredictable times. Without a way to control time, testing `debounceTime(300)` means actually waiting 300ms.

### `TestScheduler` and Marble Diagrams

RxJS provides `TestScheduler` — a virtual time scheduler that lets you compress all timing into synchronous assertions. **Marble notation** is a string-based DSL for describing Observable behaviour:

```
'-a-b-c-|'   — emit a at 10ms, b at 30ms, c at 50ms, complete at 70ms
'--a--#'     — emit a at 20ms, error at 50ms
'----a'      — emit a at 40ms, never completes
'|'          — complete immediately
'#'          — error immediately
'(abc|)'     — emit a, b, c and complete synchronously in one frame
```

Each `-` represents 10ms of virtual time.

### Example Test

```typescript
import { TestScheduler } from 'rxjs/testing';
import { debounceTime, map } from 'rxjs/operators';
import { expect } from 'vitest';

const scheduler = new TestScheduler((actual, expected) => {
  expect(actual).toEqual(expected);
});

test('debounceTime(30) suppresses rapid emissions', () => {
  scheduler.run(({ cold, expectObservable }) => {
    const source$  = cold('a-b-c------d|');   // rapid a,b,c then silence then d
    const expected =      '----------c----(d|)'; // only c and d survive debounce

    const result$ = source$.pipe(debounceTime(30, scheduler));

    expectObservable(result$).toBe(expected);
  });
});
```

### Testing Higher-Order Operators

```typescript
test('switchMap cancels previous inner observable', () => {
  scheduler.run(({ cold, hot, expectObservable }) => {
    const outer$ = hot('  --a--b---------|');
    const inner  = {
      a: cold('          --x--y--|'),     // a's inner: x at 20ms, y at 40ms
      b: cold('                --p--|')   // b's inner: p at 20ms after b
    };

    const expected = '    ----x-----p----|';
    // x arrives (from a's inner), then b arrives and cancels a's inner,
    // then p arrives from b's inner

    const result$ = outer$.pipe(
      switchMap(letter => inner[letter as keyof typeof inner])
    );

    expectObservable(result$).toBe(expected);
  });
});
```

### Why Marble Testing Matters

- Tests run **synchronously** — no `setTimeout`, no `await`, no flakiness.
- Time is **compressed** — 5 seconds of virtual time runs in microseconds.
- Operator behaviour is **precisely asserted** — not just "eventually emits something" but "emits x at exactly 40ms".
- Tests are **readable** — the marble string is a visual timeline that documents the expected behaviour.

---

## 5. Multicasting: Avoiding Duplicate Executions

When a cold Observable (such as an HTTP request) is subscribed to multiple times, each subscription triggers a new execution. This is usually undesired and a source of hard-to-debug bugs:

```typescript
const user$ = ajax.getJSON<User>('/api/user'); // cold

// This fires TWO HTTP requests:
user$.subscribe(renderHeader);
user$.subscribe(renderSidebar);
```

Fix with `shareReplay(1)`:

```typescript
const user$ = ajax.getJSON<User>('/api/user').pipe(
  shareReplay(1) // one request, both subscribers receive the same response
);

user$.subscribe(renderHeader);
user$.subscribe(renderSidebar);
```

`shareReplay(1)` is the single most common practical fix in RxJS code. Whenever a source Observable is subscribed to in more than one place and the source should execute only once, add `shareReplay(1)`.
