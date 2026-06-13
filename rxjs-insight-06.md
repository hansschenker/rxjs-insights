# Insight 06 — Functional Reactive Programming (FRP)

> Part of [rxjs-insight-groups.md](rxjs-insight-groups.md)  
> Primary sources: [rxjs-insights-claude-01.txt](rxjs-insights-claude-01.txt), [rxjs-essentials-vs-javascript-essentials.txt](rxjs-essentials-vs-javascript-essentials.txt), [rxjs-insights-grok-4.txt](rxjs-insights-grok-4.txt), [rxjs-insights-gemini.txt](rxjs-insights-gemini.txt)

---

## What FRP Is

**Functional Reactive Programming** is the synthesis of two paradigms:

- **Functional Programming (FP)** — pure functions, immutability, function composition, referential transparency.
- **Reactive Programming (RP)** — asynchronous data streams, propagation of change, push-based delivery.

Neither alone is sufficient. FP without RP gives you excellent tools for static transformations but no model for time-varying values. RP without FP gives you event streams but no disciplined way to compose transformations over them.

FRP combines them: **describe time-varying values using pure, composable functions**.

---

## Contributions from Functional Programming

### Pure Functions as Operators

Every RxJS operator is a **pure function** — same input always produces the same output, no side effects:

```typescript
// Pure: no hidden state, no side effects, deterministic output
const double = map((x: number) => x * 2);
const onlyEvens = filter((x: number) => x % 2 === 0);

// Compose them — the composition is also pure
const evenDoubles = pipe(onlyEvens, double);

// Apply to any stream — the behaviour is guaranteed
stream1$.pipe(evenDoubles);
stream2$.pipe(evenDoubles); // Same transformation, different source
```

Purity means operators can be:
- Tested in isolation.
- Reused across streams without risk.
- Reasoned about without knowing what else is happening in the application.

### Immutability

Operators never modify the source Observable. Each operator returns a **new** Observable:

```typescript
const source$ = from([1, 2, 3]);

// source$ is unchanged
const doubled$ = source$.pipe(map(x => x * 2));
const filtered$ = source$.pipe(filter(x => x > 1));

source$.subscribe(x => console.log('original:', x));  // 1, 2, 3
doubled$.subscribe(x => console.log('doubled:', x));  // 2, 4, 6
filtered$.subscribe(x => console.log('filtered:', x)); // 2, 3
```

This mirrors how `Array.map()` returns a new array without touching the original.

### Function Composition

Complex pipelines are built by composing small, single-purpose functions:

```typescript
// Small, reusable, named pieces
const debounce300 = debounceTime(300);
const toUpperCase = map((s: string) => s.toUpperCase());
const maxLength20 = filter((s: string) => s.length <= 20);
const addPrefix   = (prefix: string) => map((s: string) => `${prefix}: ${s}`);

// Compose into a pipeline
const processInput = pipe(
  debounce300,
  toUpperCase,
  maxLength20,
  addPrefix('USER')
);

// Reuse across different sources
searchBox$.pipe(processInput);
commandLine$.pipe(processInput);
```

The `pipe()` function is function composition made readable left-to-right (rather than the right-to-left nesting of traditional mathematical function composition).

### Referential Transparency

An expression is **referentially transparent** if it can be replaced by its value without changing program behaviour. In RxJS this means you can extract any sub-pipeline into a named variable without consequence:

```typescript
// These are behaviourally identical
const stream1$ = interval(1000).pipe(
  map(x => x * 2),
  filter(x => x < 10)
);

const doubled = map((x: number) => x * 2);
const lessThan10 = filter((x: number) => x < 10);

const stream2$ = interval(1000).pipe(doubled, lessThan10);
```

Referential transparency enables safe, mechanical refactoring.

### Lazy Evaluation

An Observable definition does nothing until subscribed. The computation is **lazy**:

```typescript
// This defines the computation but does not run it
const expensive$ = source$.pipe(
  map(x => {
    console.log('Computing…'); // Will NOT log yet
    return heavyComputation(x);
  })
);

// Nothing has happened yet

const sub = expensive$.subscribe(result => {
  // NOW the computation runs
  console.log(result);
});

sub.unsubscribe(); // Computation stops
```

Laziness means:
- Streams are cheap to define.
- You pay only for what you subscribe to.
- Unsubscribing cancels work in progress.

---

## Contributions from Reactive Programming

### Behaviours and Events

FRP distinguishes two kinds of time-varying values:

**Behaviour** — a value that *always exists* and varies continuously over time. Modelled in RxJS as `BehaviorSubject` or any stream combined with `startWith` + `shareReplay(1)`:

```typescript
// mousePosition$ always has a current value
const mousePosition$ = fromEvent<MouseEvent>(document, 'mousemove').pipe(
  map(e => ({ x: e.clientX, y: e.clientY })),
  startWith({ x: 0, y: 0 }),
  shareReplay(1)
);
```

**Event** — a value that exists only at discrete moments in time. Modelled as a plain Observable that emits on demand:

```typescript
// clicks$ only has a value when someone clicks
const clicks$ = fromEvent(button, 'click');
```

Combining behaviours and events is a powerful pattern:

```typescript
// Sample the current mouse position at the moment of each click
const clickPositions$ = clicks$.pipe(
  withLatestFrom(mousePosition$), // withLatestFrom samples a behaviour at an event's T
  map(([, pos]) => pos)
);
```

### Time as a First-Class Value

FRP makes time explicit and manipulable, not an invisible concern hidden inside callbacks:

```typescript
// Add timing metadata to every emission
const withTiming$ = source$.pipe(
  timestamp(),
  map(({ value, timestamp }) => ({ value, time: timestamp }))
);

// Measure inter-emission intervals
const intervals$ = source$.pipe(timeInterval());

// Replay past emissions
const history$ = source$.pipe(
  scan((history, value) => [...history, value].slice(-100), [] as number[]),
  shareReplay(1)
);
```

---

## Side Effects at the Edges

The most important discipline in FRP is **isolating side effects**. The pipeline should be pure; effects happen only at the boundary:

```typescript
// Pure pipeline — no side effects anywhere in here
const validated$ = userInput$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  map(validateInput),
  filter(isValid),
  map(transformData)
);

// Side effects only in subscribe (the sink/edge)
validated$.subscribe({
  next: data => {
    updateDOM(data);           // side effect
    saveToLocalStorage(data);  // side effect
    sendAnalytics(data);       // side effect
  }
});
```

When debugging requires a side effect inside the pipeline, use `tap` — which performs a side effect but passes the value through unchanged:

```typescript
const piped$ = source$.pipe(
  map(x => x * 2),
  tap(x => console.log('after map:', x)), // side effect for debugging only
  filter(x => x > 5)
);
```

`tap` is the FRP equivalent of a breakpoint: it lets you observe without disturbing.

---

## The FRP Architecture for State Management

The canonical FRP state pattern in RxJS:

```typescript
// Actions as an event stream
const action$ = new Subject<Action>();

// State as a behaviour derived from actions
const state$ = action$.pipe(
  scan(reducer, initialState), // pure reducer: (State, Action) => State
  startWith(initialState),
  shareReplay(1)
);

// Derived state via pure functions
const user$ = state$.pipe(map(s => s.user), distinctUntilChanged());
const isLoggedIn$ = user$.pipe(map(u => !!u));
const itemCount$ = state$.pipe(map(s => s.items.length), distinctUntilChanged());

// Dispatch is the only way to cause state change
const dispatch = (action: Action) => action$.next(action);

// Effects: action in → action out, no state mutation
const loginEffect$ = action$.pipe(
  filter(a => a.type === 'LOGIN'),
  exhaustMap(a => authService.login(a.payload).pipe(
    map(user => ({ type: 'LOGIN_SUCCESS', user })),
    catchError(err => of({ type: 'LOGIN_FAILED', err }))
  ))
);
loginEffect$.subscribe(dispatch);
```

This pattern (sometimes called MVU or Elm Architecture in RxJS) is the purest expression of FRP in a web application:
- State is derived, never mutated.
- All changes flow through a single pure reducer.
- Effects are isolated and return new actions, never touching state directly.
- The entire system is a data flow graph with a single source of truth.

---

## FRP vs Reactive Programming

| | Reactive Programming | Functional Reactive Programming |
|---|---|---|
| Operator purity | Not required | Required — operators are pure functions |
| Immutability | Not required | Required — new Observables, not mutations |
| Side effects | Can appear anywhere | Isolated to `subscribe` / `tap` |
| Composability | Possible | Guaranteed by mathematical laws |
| Testability | Varies | High — pure functions are trivially testable |
