# Insight 08 — Unifying Async Programming Model

> Part of [rxjs-insight-groups.md](rxjs-insight-groups.md)  
> Primary sources: [rxjs-insights-claude-01.txt](rxjs-insights-claude-01.txt), [rxjs-fundamental-insights-chatgpt-45-version-2-refined.txt](rxjs-fundamental-insights-chatgpt-45-version-2-refined.txt), [rxjs-insights-list.txt](rxjs-insights-list.txt)

---

## The Problem: JavaScript's Fragmented Async Landscape

Before RxJS, JavaScript had at least five incompatible async patterns, each with its own API surface, its own cleanup mechanism, and its own error model. Developers working on non-trivial apps had to hold all of them in their head simultaneously and coordinate between them manually.

### 1. Callbacks

```typescript
// Error-first Node style
fs.readFile('file.txt', (err, data) => { /* … */ });
// Or legacy
setTimeout(() => { /* … */ }, 1000);
```

No cancellation. No standard error propagation. No composition. Completion is implicit.

### 2. Promises

```typescript
fetch('/api').then(res => res.json()).catch(err => /* … */);
```

Single value only. No cancellation. Errors escape to unhandled rejection if not caught. Cannot represent a stream of values over time.

### 3. Node EventEmitters

```typescript
emitter.on('data', handler);
emitter.on('error', errHandler);
emitter.on('end', endHandler);
emitter.removeListener('data', handler); // manual cleanup
```

Multiple values. Manual subscription management. No composition primitives.

### 4. DOM Events

```typescript
element.addEventListener('click', handler);
element.removeEventListener('click', handler); // manual cleanup
```

Similar to EventEmitter but different API. No composition.

### 5. Node Streams / WebSockets / SSE

Each has its own interface, its own backpressure semantics, and its own lifecycle management.

**The result**: complex applications required manual coordination between incompatible systems, with ad-hoc glue code, scattered cleanup logic, and no shared vocabulary for transformation.

---

## RxJS's Solution: One Interface for All

RxJS wraps every async source behind the same `Observable` interface:

| Source | RxJS bridge |
|---|---|
| Node callback | `bindNodeCallback(fn)` |
| Plain callback | `bindCallback(fn)` |
| Promise | `from(promise)` |
| DOM event | `fromEvent(element, eventName)` |
| Node EventEmitter | `fromEventPattern(addHandler, removeHandler)` |
| Iterable / Array | `from(iterable)` |
| Single value | `of(value)` |
| Timer | `timer(delay)` |
| Interval | `interval(ms)` |
| WebSocket | `webSocket(url)` |
| AJAX | `ajax(config)` |

Once wrapped, they are all just Observables.

---

## What Unification Gives You

### Same Operators on Every Source

The same `map`, `filter`, `debounceTime`, `switchMap`, `retry`, `catchError` work on any Observable, regardless of its underlying source:

```typescript
// A reusable processing pipeline
const processAny = <T>(source$: Observable<T>) => source$.pipe(
  filter(x => x != null),
  debounceTime(100),
  distinctUntilChanged(),
  catchError(err => of(null))
);

// Works identically on all sources
processAny(fromEvent(input, 'input'));
processAny(from(fetch('/api')));
processAny(interval(1000));
processAny(webSocket('wss://feed'));
```

### Same Error Handling

```typescript
// One error handling strategy for everything
source$.pipe(
  retry(3),
  catchError(err => {
    logError(err);
    return of(defaultValue);
  })
);
```

Before RxJS: callback errors use `if (err)`, Promises use `.catch()`, EventEmitters use `.on('error', …)`, each requiring separate handling logic.

### Same Cancellation Mechanism

```typescript
// One cancellation mechanism for everything
const sub = anySource$.subscribe(handler);

// Cancels the underlying source regardless of what it is:
// removes event listeners, aborts fetch, clears timers
sub.unsubscribe();
```

### Same Composition Primitives

Different sources can be combined using the same operators:

```typescript
// Combine an HTTP request, a timer, and a DOM event
const combined$ = combineLatest([
  from(fetch('/api/user')),
  interval(5000),
  fromEvent(document, 'visibilitychange')
]).pipe(
  filter(([_, __, event]) => document.visibilityState === 'visible'),
  switchMap(() => from(fetch('/api/user')))
);
```

Without RxJS this coordination requires managing three separate APIs, three separate cleanup mechanisms, and hand-written synchronisation state.

---

## Observable as a Universal Interface

The Observable becomes a **universal data source interface**. A service returning data can switch its underlying implementation without changing the consumer:

```typescript
class DataService {
  getData(): Observable<Data> {
    // Any of these can be returned — the consumer's code is unchanged:

    return from(fetch('/api/data'));           // HTTP
    return fromEvent(ws, 'message');           // WebSocket
    return interval(5000).pipe(               // Polling
      switchMap(() => from(fetch('/api')))
    );
    return of(cachedData);                    // Synchronous cached
    return timer(1000).pipe(map(() => data)); // Delayed static
  }
}
```

The consumer subscribes and processes values — it does not need to know which async mechanism is in use.

---

## The Before / After Comparison

### Before RxJS: Coordinating Multiple Async Patterns

```typescript
// Click → HTTP request → WebSocket → timeout
// Three incompatible APIs, manual cleanup, fragile coordination

let wsHandler: (e: MessageEvent) => void;
let timeoutId: number;

button.addEventListener('click', async () => {
  clearTimeout(timeoutId);

  const res = await fetch('/api/session');

  ws.addEventListener('message', wsHandler = (e) => {
    processMessage(JSON.parse(e.data));
  });

  timeoutId = window.setTimeout(() => {
    ws.removeEventListener('message', wsHandler);
  }, 5000);
});

// Cleanup — must remember every handle
button.removeEventListener('click', /* what was the handler? */);
ws.removeEventListener('message', wsHandler);
clearTimeout(timeoutId);
```

### After RxJS: One Unified Pipeline

```typescript
const workflow$ = fromEvent(button, 'click').pipe(
  switchMap(() => from(fetch('/api/session'))),
  switchMap(() => fromEvent(ws, 'message').pipe(
    map(e => JSON.parse((e as MessageEvent).data)),
    timeout(5000),
    takeUntil(fromEvent(button, 'stop'))
  ))
);

const sub = workflow$.subscribe(processMessage);

// One line cleans up everything
sub.unsubscribe();
```

---

## Makes Events First-Class Citizens

A key philosophical shift: in standard JavaScript, event handlers are callbacks passed as arguments — they are not values you can store, transform, or pass around.

RxJS **promotes events to first-class values** by wrapping them in Observables:

```typescript
// An event stream is just a value — store it, pass it, compose it
const clicks$: Observable<MouseEvent> = fromEvent(button, 'click');
const keydowns$: Observable<KeyboardEvent> = fromEvent(document, 'keydown');

// Pass it to a function
processStream(clicks$);

// Compose it with other streams
const bothInputs$ = merge(clicks$, keydowns$);
```

This is the same shift that functional programming made when it promoted functions to first-class values. Observable does the same for event sequences.
