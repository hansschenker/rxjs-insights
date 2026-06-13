# Insight 10 — Three-Step Workflow: Source → Pipeline → Sink

> Part of [rxjs-insight-groups.md](rxjs-insight-groups.md)  
> Primary sources: [rxjs-insights-list.txt](rxjs-insights-list.txt), [rxjs-insights-grok-4.txt](rxjs-insights-grok-4.txt), [rxjs-insights-gemini.txt](rxjs-insights-gemini.txt), [rxjs-insights-grok-4-notebooklm.txt](rxjs-insights-grok-4-notebooklm.txt)

---

## The Canonical Shape

Every RxJS program, regardless of complexity, has the same three-part structure:

```
Source  →  Pipeline  →  Sink
Create  →  Pipe      →  Subscribe
```

1. **Create** — produce an Observable (enter the RxJS world).
2. **Pipe** — transform the Observable through a chain of operators.
3. **Subscribe** — consume the result and perform side effects (exit the RxJS world).

This is not a convention — it is the architecture enforced by the Observable API itself.

---

## Step 1 — Create (Source)

The source is where you enter the RxJS world by creating an Observable. Every piece of data or event you want to work with must first be wrapped in an Observable.

Creation operators are the vocabulary for this step:

```typescript
// From synchronous values
const nums$ = of(1, 2, 3);
const arr$  = from([1, 2, 3]);

// From time
const tick$ = interval(1000);
const once$ = timer(2000);

// From external events
const clicks$ = fromEvent(button, 'click');
const input$  = fromEvent<InputEvent>(inputEl, 'input');

// From async operations
const http$ = ajax.getJSON<User[]>('/api/users');
const ws$   = webSocket<Message>('wss://server/feed');
const p$    = from(fetch('/api/data').then(r => r.json()));

// Programmatic
const custom$ = new Observable<number>(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.complete();
});
```

The source defines **what** data flows into the pipeline and **when** it arrives.

---

## Step 2 — Pipe (Pipeline)

The pipeline is the transformation stage. It takes the Observable produced in step 1 and chains operators over it, each producing a new Observable. The pipeline itself is **pure and lazy** — it describes transformations without executing them.

```typescript
const result$ = clicks$.pipe(      // Observable<MouseEvent>
  throttleTime(1000),              // Observable<MouseEvent> — rate-limited
  map(e => e.clientX),             // Observable<number>    — x coordinate
  filter(x => x > 200),           // Observable<number>    — filtered
  scan((acc, x) => acc + x, 0),   // Observable<number>    — running sum
  distinctUntilChanged()           // Observable<number>    — deduplicated
);
```

The types flow through the pipeline and TypeScript narrows them at each step — `map(e => e.clientX)` transforms `Observable<MouseEvent>` to `Observable<number>`, and subsequent operators receive `number`.

### The Pipeline Is Lazy

Defining a pipeline executes nothing:

```typescript
// Nothing happens here — no event listeners are attached, no HTTP requests are made
const search$ = fromEvent(inputEl, 'input').pipe(
  debounceTime(300),
  map(e => (e.target as HTMLInputElement).value),
  switchMap(q => ajax.getJSON<Result[]>(`/api/search?q=${q}`))
);

// Still nothing has happened
```

The pipeline is a **blueprint** — a description of what to do when data eventually arrives.

### Operators Create a Tree of Subscriptions

Behind the scenes, each operator in a pipeline becomes a node in a **subscription tree**. This tree only comes alive when step 3 (subscribe) happens, and it tears down when the subscription is disposed.

---

## Step 3 — Subscribe (Sink)

Subscribing **activates** the pipeline. This is where:
- Event listeners are attached (for `fromEvent`).
- HTTP requests are initiated (for `ajax`).
- Timers are started (for `interval`, `timer`).
- Side effects happen.

```typescript
// Nothing was happening before this line
const subscription = result$.subscribe({
  next: value => updateUI(value),    // called for each emitted value
  error: err  => showError(err),     // called once on error; stream ends
  complete: () => console.log('done') // called once on completion; stream ends
});
```

The Observer passed to `subscribe` is the **sink** — the consumer of the stream. It is the only place in an RxJS program where side effects are expected and intentional.

### Shorthand Subscription

If you only care about values (no error or completion handling):

```typescript
result$.subscribe(value => updateUI(value));
```

### The Subscription Object

`subscribe()` returns a `Subscription`:

```typescript
const sub = result$.subscribe(handler);

// Later — tear down the entire pipeline
sub.unsubscribe();
```

`unsubscribe()` reverses everything activated in step 3:
- Removes event listeners.
- Cancels pending HTTP requests (where possible).
- Clears timers.
- Frees memory held by the subscription tree.

---

## The Boundary Between Worlds

The three-step structure creates a clean boundary:

```
┌──────── RxJS World ─────────┐   ┌─── Side-Effect World ───┐
│                              │   │                          │
│  Create ──→ Pipe ──→ (pipe)  │   │  Subscribe              │
│                              │   │    next(value)           │
│  Pure, lazy, composable      │   │    error(err)            │
│  No side effects             │   │    complete()            │
│                              │   │                          │
└──────────────────────────────┘   └──────────────────────────┘
```

Inside the pipe: pure transformations, no effects, referentially transparent.  
At subscribe: controlled side effects, one entry point per stream.

This boundary is what makes RxJS code testable and reasoning about it tractable — you can understand the pipeline independently of what its consumer does with the values.

---

## The Subscription Tree in Action

When you subscribe, the subscription propagates backward through the pipeline:

```
subscribe(observer)
  → filter subscribes to map
    → map subscribes to throttleTime
      → throttleTime subscribes to fromEvent
        → fromEvent attaches event listener   ← activation happens here
```

When you unsubscribe:

```
subscription.unsubscribe()
  → filter unsubscribes from map
    → map unsubscribes from throttleTime
      → throttleTime unsubscribes from fromEvent
        → fromEvent removes event listener   ← cleanup happens here
```

The subscription tree is invisible to the developer — it is managed entirely by the RxJS internals. But understanding it clarifies why `unsubscribe()` is so important: it is the only way to trigger the full cleanup chain.

---

## Source → Pipeline → Sink as a Mental Model

This model is powerful because it forces a clean separation of concerns:

| Step | Question answered | Concerns |
|---|---|---|
| **Source** | Where does data come from? | Data origin, timing |
| **Pipeline** | What transformation does the data undergo? | Business logic, transformation |
| **Sink** | What happens with the result? | Side effects, UI updates, storage |

When debugging, this structure tells you where to look:
- Wrong values? The bug is in the **pipeline**.
- Nothing arriving? The bug is in the **source** (or subscription not triggered).
- Wrong effects? The bug is in the **sink**.
- Memory leak? A **subscription** was never unsubscribed.
