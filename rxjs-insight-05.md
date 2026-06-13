# Insight 05 — Reactive Programming Paradigm

> Part of [rxjs-insight-groups.md](rxjs-insight-groups.md)  
> Primary sources: [rxjs-insights-claude-01.txt](rxjs-insights-claude-01.txt), [rxjs-insights-grok-4.txt](rxjs-insights-grok-4.txt), [rxjs-insights-gemini.txt](rxjs-insights-gemini.txt), [rxjs-essentials-vs-javascript-essentials.txt](rxjs-essentials-vs-javascript-essentials.txt)

---

## What Reactive Programming Is

Reactive programming is a programming paradigm organised around **asynchronous data streams and the automatic propagation of change**. Instead of writing instructions that say *how* to update state, you declare *what depends on what* and let the system propagate changes automatically.

The word "reactive" refers to the system's relationship to incoming data: rather than polling for change or explicitly invoking updates, components *react* to changes as they arrive.

---

## The Core Shift: From Imperative State to Declarative Data Flow

In imperative programming you manage state manually:

```typescript
// Imperative: you coordinate every update by hand
let count = 0;
let doubled = 0;

button.onclick = () => {
  count++;
  doubled = count * 2;  // manually keep doubled in sync
  countDisplay.textContent = String(count);
  doubledDisplay.textContent = String(doubled);
};
```

In reactive programming you declare the relationships and let change propagate:

```typescript
// Reactive: declare dependencies, propagation is automatic
const count$ = fromEvent(button, 'click').pipe(
  scan(acc => acc + 1, 0),
  startWith(0)
);

const doubled$ = count$.pipe(map(n => n * 2));

count$.subscribe(n => countDisplay.textContent = String(n));
doubled$.subscribe(n => doubledDisplay.textContent = String(n));
```

`doubled$` does not need to be told when `count$` changes — it reacts automatically because it declared a dependency on `count$`. This is the essence of reactive programming.

---

## The Dependency Graph

Every RxJS stream composition builds an implicit **directed acyclic graph (DAG)** of data dependencies. Each node is an Observable; each edge is a subscription.

```typescript
const price$    = new BehaviorSubject(100);
const quantity$ = new BehaviorSubject(5);
const taxRate$  = new BehaviorSubject(0.1);

const subtotal$ = combineLatest([price$, quantity$]).pipe(
  map(([p, q]) => p * q)
);

const tax$ = combineLatest([subtotal$, taxRate$]).pipe(
  map(([s, r]) => s * r)
);

const total$ = combineLatest([subtotal$, tax$]).pipe(
  map(([s, t]) => s + t)
);
```

The dependency graph:

```
price$    ──┐
            ├──→ subtotal$ ──┬──→ total$
quantity$ ──┘                │
                             │
taxRate$  ──────→ tax$  ─────┘
```

Changing `price$` automatically propagates: `price$ → subtotal$ → tax$ → total$`. No manual notification required.

### What the Graph Implies

- **No stale data**: nodes always reflect the current state of their inputs.
- **No manual synchronisation**: dependencies are structural, not procedural.
- **Lazy activation**: the graph is only active while there is a subscriber at the leaves. Remove all subscriptions and the entire graph shuts down automatically.
- **Memory safety**: unsubscribing from a leaf tears down the chain above it, preventing retained closures and event listeners.

---

## The Reactive Manifesto: Four Characteristics

The [Reactive Manifesto](https://www.reactivemanifesto.org) defines reactive *systems* (not just libraries) by four properties. RxJS provides operators that implement each of them:

### 1. Responsive

A responsive system reacts to inputs within a bounded time. In RxJS:

```typescript
const search$ = input$.pipe(
  debounceTime(300),              // cap response latency
  switchMap(query => api.search(query).pipe(
    timeout(5000),                // guarantee maximum wait time
    catchError(() => of([]))      // graceful degradation on timeout
  ))
);
```

### 2. Resilient

A resilient system stays responsive under failure. In RxJS:

```typescript
const data$ = source$.pipe(
  retry(3),                                // retry up to 3 times
  catchError(err => fallbackSource$),      // switch to fallback
  catchError(err => of(defaultValue))      // last resort default
);
```

### 3. Elastic

An elastic system remains responsive under varying load. In RxJS:

```typescript
const processed$ = highFrequencySource$.pipe(
  bufferTime(100),                          // absorb bursts
  filter(batch => batch.length > 0),
  mergeMap(batch => processBatch(batch), 4) // limit concurrency to 4
);
```

### 4. Message-Driven

Components communicate via asynchronous messages, not direct method calls. In RxJS:

```typescript
const messageBus$ = new Subject<Message>();

// Component A publishes
componentA.events$.subscribe(event =>
  messageBus$.next({ type: 'EVENT_A', payload: event })
);

// Component B subscribes — no direct reference to A
const aEvents$ = messageBus$.pipe(
  filter(msg => msg.type === 'EVENT_A'),
  map(msg => msg.payload)
);
```

This decouples components: neither knows about the other. They communicate only through the shared message bus.

---

## Propagation of Change: What "Reactive" Actually Means

"Propagation of change" is not just a metaphor — it is a precise description of what happens inside an RxJS subscription graph. When a source emits:

1. The emission enters the subscription tree at the source node.
2. Each operator node processes it and forwards the result downstream.
3. The sink (subscriber) receives the final transformed value.

All of this happens synchronously within a single event-loop tick for synchronous operators, or asynchronously for time-based operators. The developer writes none of this propagation logic — it is entirely handled by the operator chain.

### Diamond Dependencies

When two branches of the graph both depend on the same root, and both feed into a single sink, a "diamond" topology forms:

```
      root$
      /    \
   left$  right$
      \    /
     bottom$
```

RxJS handles this correctly: when `root$` emits, `bottom$` receives *one* update reflecting both `left$` and `right$`'s latest values, not two intermediate updates. This avoids "glitches" — momentary inconsistent states that would occur if `bottom$` updated after `left$` alone before `right$` had processed the new value.

---

## Asynchronous Data Streams as First-Class Values

The paradigm shift reactive programming demands is treating asynchronous sequences as first-class values that can be:

- **Passed around** — an Observable is just a value you can store in a variable.
- **Transformed** — operators return new Observables without modifying the original.
- **Combined** — `combineLatest`, `merge`, `zip` compose multiple streams into one.
- **Subscribed to multiply** — multiple consumers can observe the same stream independently (or share it via multicasting).

This is analogous to how functional programming treats functions as first-class values. Reactive programming does the same for time-varying sequences.

---

## Reactive vs Event-Driven

Reactive programming is often confused with event-driven programming. They overlap but are distinct:

| | Event-Driven | Reactive |
|---|---|---|
| **Focus** | Individual events and their handlers | Streams and their transformations |
| **State management** | Usually imperative/manual | Declarative through stream composition |
| **Composition** | Hard — handlers are isolated | Natural — operators compose |
| **Cancellation** | Manual (removeEventListener) | Automatic via unsubscription |
| **Multiple sources** | Requires explicit coordination | `combineLatest`, `merge`, etc. |

RxJS subsumes event-driven programming: DOM events become Observables via `fromEvent`, and from that point forward the reactive paradigm applies uniformly.
