# Insight 14 — TypeScript Integration

> Part of [rxjs-insight-groups.md](rxjs-insight-groups.md)  
> Primary sources: [rxjs-insights-grok-4.txt](rxjs-insights-grok-4.txt), [rxjs-insights-gemini.txt](rxjs-insights-gemini.txt), [Rxjs-insights-summariy-0-evolution.txt](Rxjs-insights-summariy-0-evolution.txt), [rxjs-insights-grok-4-notebooklm.txt](rxjs-insights-grok-4-notebooklm.txt)

---

## Why TypeScript Matters for RxJS

RxJS existed before TypeScript. In its early JavaScript form it was powerful but treacherous: a `map` that returned the wrong type would fail at runtime, often buried deep inside an async callback. Refactoring a pipeline meant trusting your memory.

TypeScript changed this. With type definitions, RxJS pipelines become **compile-time-checked data flow programs**. The type of each emission is tracked through every operator, and a type error in a pipeline is caught by the editor before the code ever runs.

This is the same progression RxJS itself made historically: just as Haskell's type system gave compile-time guarantees to functional programs (see Insight 01), TypeScript gives compile-time guarantees to RxJS pipelines.

---

## The Core Types

### `Observable<T>`

The fundamental type. `T` is the type of values emitted:

```typescript
const numbers$: Observable<number> = of(1, 2, 3);
const users$: Observable<User> = ajax.getJSON<User>('/api/user');
const events$: Observable<MouseEvent> = fromEvent<MouseEvent>(document, 'click');
```

TypeScript tracks `T` through the pipeline:

```typescript
const doubled$: Observable<number> = numbers$.pipe(map(x => x * 2));
//                         ^^^^^^ inferred — TypeScript knows x is number
```

### `Observer<T>`

The consumer interface. All three methods are typed:

```typescript
interface Observer<T> {
  next(value: T): void;
  error(err: unknown): void;
  complete(): void;
}

const observer: Observer<number> = {
  next: (x) => console.log(x),   // x is number — TypeScript enforces this
  error: (e) => console.error(e),
  complete: () => console.log('done')
};
```

### `Subscription`

The lifecycle handle returned by `subscribe()`:

```typescript
const sub: Subscription = source$.subscribe(observer);
sub.unsubscribe();

// Subscriptions can be composed
const group = new Subscription();
group.add(sub1);
group.add(sub2);
group.unsubscribe(); // unsubscribes both
```

### `Subject<T>` and Variants

```typescript
const action$ = new Subject<Action>();        // hot, no initial value
const state$  = new BehaviorSubject<State>(initialState); // always has current value
const log$    = new ReplaySubject<LogEntry>(100);          // replays last 100

// Type is enforced on both push and pull sides
action$.next({ type: 'CLICK', payload: 42 }); // must match Action type
action$.subscribe((a: Action) => handle(a));   // a is typed as Action
```

---

## `OperatorFunction<T, R>`: The Key to Typed Pipelines

`OperatorFunction<T, R>` is the type of any pipeable operator — a function that takes `Observable<T>` and returns `Observable<R>`:

```typescript
type OperatorFunction<T, R> = (source: Observable<T>) => Observable<R>;
```

This type is what makes TypeScript's type inference work through `pipe()`. Each operator declares what it consumes and what it produces, and TypeScript chains them:

```typescript
const source$: Observable<string> = fromEvent<InputEvent>(input, 'input').pipe(
  map((e: InputEvent) => (e.target as HTMLInputElement).value)
);
//   Observable<string> from here onward

const result$: Observable<number> = source$.pipe(
  filter((s: string) => s.length > 2),   // Observable<string> → Observable<string>
  map((s: string) => s.length),          // Observable<string> → Observable<number>
  distinctUntilChanged()                 // Observable<number> → Observable<number>
);
```

If you make a type error — passing `Observable<string>` to an operator that expects `Observable<number>` — TypeScript reports the error at the `pipe()` call site, not at runtime.

### Custom Operators with `OperatorFunction<T, R>`

Typing a custom operator is straightforward:

```typescript
// A custom operator that only passes values matching a type guard
function filterType<T, R extends T>(
  guard: (value: T) => value is R
): OperatorFunction<T, R> {
  return (source: Observable<T>): Observable<R> =>
    source.pipe(
      filter(guard)
    ) as Observable<R>;
}

// Usage — TypeScript narrows the type
const strings$: Observable<string> = mixed$.pipe(
  filterType((x): x is string => typeof x === 'string')
);
```

Without `OperatorFunction<T, R>`, custom operators require casting and provide no type safety. With it, the type flows through just like a built-in operator.

---

## Type Inference Through `pipe()`

RxJS overloads `pipe()` with up to 9 type parameters (one per operator), allowing TypeScript to infer the output type of a chain:

```typescript
const result$ = of(1, 2, 3).pipe(
  map(x => x * 2),          // OperatorFunction<number, number>
  filter(x => x > 3),       // OperatorFunction<number, number>
  map(x => `${x}px`),       // OperatorFunction<number, string>
  take(5)                    // OperatorFunction<string, string>
);
// TypeScript infers: Observable<string>
```

Each step in the chain is independently typed. The compiler verifies that the output type of each operator is compatible with the input type of the next.

---

## TypeScript and Higher-Order Operators

Higher-order operators (`mergeMap`, `switchMap`, `concatMap`, `exhaustMap`) are particularly important to type correctly because they involve a function that returns an Observable:

```typescript
// TypeScript infers the inner and outer types
const users$: Observable<User> = userId$.pipe(
  mergeMap((id: number) => ajax.getJSON<User>(`/api/users/${id}`))
  // mergeMap<number, User>
);

// Chaining higher-order operators
const comments$: Observable<Comment[]> = userId$.pipe(
  mergeMap((id: number) => ajax.getJSON<User>(`/api/users/${id}`)),
  // Observable<User>
  mergeMap((user: User) => ajax.getJSON<Post[]>(`/api/posts?userId=${user.id}`)),
  // Observable<Post[]>
  mergeMap((posts: Post[]) => forkJoin(
    posts.map(p => ajax.getJSON<Comment[]>(`/api/comments?postId=${p.id}`))
  ))
  // Observable<Comment[][]>
);
```

TypeScript catches the case where you return the wrong type from the mapping function — e.g., returning a plain `User` instead of `Observable<User>` — at compile time.

---

## `MonoTypeOperatorFunction<T>`

A special case: when an operator returns the **same type** it receives, it uses `MonoTypeOperatorFunction<T>`:

```typescript
type MonoTypeOperatorFunction<T> = OperatorFunction<T, T>;

// These are MonoTypeOperatorFunctions: they don't change T
const deduplicate: MonoTypeOperatorFunction<number> = distinctUntilChanged();
const throttled: MonoTypeOperatorFunction<number> = throttleTime(100);
const filtered: MonoTypeOperatorFunction<number> = filter(x => x > 0);
```

This makes it clear in the type signature that an operator is a "pass-through with filtering" rather than a type transformer.

---

## TypeScript Elevated RxJS

The historical arc (from Insight 01) reaches its conclusion here: TypeScript restored the mathematical rigour that Haskell had at the beginning of the lineage, now available in the most widely used web programming language.

Specifically:
- The `Functor` contract (`map` preserves structure) is enforced: `Observable<T>.pipe(map(T => R))` yields `Observable<R>`, never anything else.
- The `Monad` contract (`flatMap` flattens) is typed: `mergeMap(T => Observable<R>)` yields `Observable<R>`, not `Observable<Observable<R>>`.
- The pipeline's type is the composition of its operator types, which TypeScript verifies mechanically.

The result: a large class of RxJS bugs — type mismatches in pipelines, missing operators, accidentally unwrapped Observables — are impossible to ship if TypeScript's `strict` mode is enabled.
