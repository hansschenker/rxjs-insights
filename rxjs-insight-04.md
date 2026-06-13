# Insight 04 — Observable as Functor / Applicative / Monad

> Part of [rxjs-insight-groups.md](rxjs-insight-groups.md)  
> Primary sources: [rxjs-insights-claude-01.txt](rxjs-insights-claude-01.txt), [Rxjs-insights-summariy-0-evolution.txt](Rxjs-insights-summariy-0-evolution.txt), [rxjs-insights-grok-4.txt](rxjs-insights-grok-4.txt), [rxjs-insights-gemini.txt](rxjs-insights-gemini.txt)

---

## Why This Matters

Observable being a Functor/Applicative/Monad is not an academic curiosity. These structures carry **mathematical laws** that guarantee:

- Operations compose predictably no matter how deeply nested.
- Refactoring is safe: equivalent expressions can be freely substituted.
- Generic abstractions (functions that work on *any* monad) apply to Observables without modification.

The elegance of RxJS pipelines is not accidental — it is the direct consequence of Observables satisfying these algebraic laws.

---

## Level 1 — Functor

A **Functor** is any container that implements `map` and satisfies two laws.

```typescript
// map :: (a -> b) -> Observable<a> -> Observable<b>
const source$ = of(5);
const mapped$ = source$.pipe(map(x => x * 2)); // Observable<10>
```

### Functor Laws

**Identity law** — mapping the identity function changes nothing:
```typescript
source$.pipe(map(x => x))
// ≡ source$
```

**Composition law** — one combined map equals two sequential maps:
```typescript
const f = (x: number) => x * 2;
const g = (x: number) => x + 1;

source$.pipe(map(x => f(g(x))))
// ≡
source$.pipe(map(g), map(f))
```

### What Functor Gives You

The composition law means you can freely merge or split `map` calls during refactoring without changing behaviour. The compiler cannot enforce this, but the mathematical guarantee allows confident transformations.

---

## Level 2 — Applicative Functor

An **Applicative Functor** extends Functor with two operations:

1. **`of` (pure/return)** — wrap a plain value into the Observable context.
2. **`ap`** — apply a wrapped function to a wrapped value.

```typescript
// of :: a -> Observable<a>
const value$ = of(5);       // Observable<5>
const fn$    = of(x => x * 2); // Observable<Function>
```

JavaScript/RxJS does not expose `ap` directly, but `combineLatest` together with `map` implements it:

```typescript
// ap :: Observable<(a -> b)> -> Observable<a> -> Observable<b>
const ap = <A, B>(fn$: Observable<(a: A) => B>) => (value$: Observable<A>) =>
  combineLatest([fn$, value$]).pipe(map(([fn, value]) => fn(value)));
```

### Lifting Functions

The key power of Applicative is **lifting** — taking a function that works on plain values and making it work on Observables:

```typescript
// liftA2: lift a binary function into Observable context
const liftA2 = <A, B, C>(fn: (a: A, b: B) => C) =>
  (a$: Observable<A>, b$: Observable<B>): Observable<C> =>
    combineLatest([a$, b$]).pipe(map(([a, b]) => fn(a, b)));

const add = (a: number, b: number) => a + b;
const sum$ = liftA2(add)(of(3), of(5)); // Observable<8>
```

`combineLatest` is the Applicative operation in practice: combine independent computations whose results need to be merged.

---

## Level 3 — Monad

A **Monad** extends Applicative with `flatMap` (also called `bind`, `>>=`, `mergeMap`):

```typescript
// flatMap :: Observable<a> -> (a -> Observable<b>) -> Observable<b>
const result$ = of(5).pipe(
  mergeMap(x => of(x * 2)),  // a -> Observable<b>
  mergeMap(x => of(x + 1))
);
// Observable<11>
```

The crucial difference from Applicative: with a Monad, **the next computation can depend on the value produced by the previous one**. This is what enables sequencing of asynchronous operations.

### The Three Monad Laws

**Left identity** — wrapping then binding equals applying directly:
```typescript
const f = (x: number) => of(x * 2);
of(5).pipe(mergeMap(f))
// ≡
f(5) // Observable<10>
```

**Right identity** — binding with `of` changes nothing:
```typescript
const m$ = of(5);
m$.pipe(mergeMap(x => of(x)))
// ≡
m$ // Observable<5>
```

**Associativity** — the order of binding does not matter, only the order of operations:
```typescript
const f = (x: number) => of(x * 2);
const g = (x: number) => of(x + 1);
const m$ = of(5);

m$.pipe(mergeMap(f), mergeMap(g))
// ≡
m$.pipe(mergeMap(x => f(x).pipe(mergeMap(g))))
// Both produce Observable<11>
```

Associativity is what makes it safe to flatten or nest `mergeMap` chains however you like during refactoring.

---

## Four Monadic Bind Variants

RxJS provides four versions of `flatMap`, each with different **scheduling semantics** but all satisfying the monad laws:

| Operator | Scheduling | Use case |
|---|---|---|
| `mergeMap` | All inner Observables run in parallel | Independent requests, parallel processing |
| `concatMap` | Inner Observables queue; next starts only after previous completes | Ordered operations, sequential uploads |
| `switchMap` | Each new value cancels the previous inner Observable | Search typeahead, live queries |
| `exhaustMap` | New values are ignored while an inner Observable is active | Prevent double-submit, login button |

All four implement the same monadic bind conceptually — they differ only in how they resolve the temporal overlap of multiple inner streams.

---

## Kleisli Composition

In category theory, a **Kleisli arrow** is a function `a → M b` — a function that returns a monadic value. RxJS functions that return Observables are Kleisli arrows:

```typescript
const fetchUser    = (id: number): Observable<User>    => ajax(`/user/${id}`);
const fetchPosts   = (user: User): Observable<Post[]>  => ajax(`/posts?userId=${user.id}`);
const fetchComment = (post: Post): Observable<Comment[]> => ajax(`/comments?postId=${post.id}`);
```

These compose via `mergeMap` (Kleisli composition):

```typescript
const userComments$ = of(userId).pipe(
  mergeMap(fetchUser),
  mergeMap(fetchPosts),
  mergeMap(posts => from(posts)),
  mergeMap(fetchComment),
  toArray()
);
```

This pattern — chaining functions that each return an Observable — is the idiomatic way to sequence dependent async operations in RxJS. It is directly analogous to Haskell's `do`-notation:

```haskell
do
  user     <- fetchUser userId
  posts    <- fetchPosts user
  comments <- mapM fetchComment posts
  return comments
```

---

## flatMap as the Universal Operator

Because `flatMap` is the monadic bind, many other RxJS operators can be expressed in terms of it. This is not just theoretical — it illustrates how central `mergeMap` is to the whole system:

```typescript
// take(n) via flatMap
const take1$ = source$.pipe(
  mergeMap((x, i) => i === 0 ? of(x) : EMPTY)
);

// filter via flatMap
const evens$ = source$.pipe(
  mergeMap(x => x % 2 === 0 ? of(x) : EMPTY)
);

// map via flatMap
const doubled$ = source$.pipe(
  mergeMap(x => of(x * 2))
);
```

Conversely, `mergeMap` is itself just `map` followed by `mergeAll`:
```typescript
source$.pipe(mergeMap(fn))
// ≡
source$.pipe(map(fn), mergeAll())
```

---

## Hierarchy Summary

```
Functor
  └─ map(fn)                         — transform values inside the Observable

Applicative Functor (extends Functor)
  ├─ of(value)                       — wrap a plain value
  └─ combineLatest + map             — apply wrapped functions to wrapped values
                                       (independent combinations)

Monad (extends Applicative)
  ├─ of(value)                       — same as Applicative's pure
  └─ mergeMap / concatMap /          — chain dependent computations
     switchMap / exhaustMap            (the next step depends on the result of the previous)
```

Each level adds expressiveness:
- Functor: transform values in a stream.
- Applicative: combine independent streams.
- Monad: sequence dependent streams.

The distinction between Applicative and Monad in practice: use `combineLatest` when computations are **independent** (both can run at the same time); use `mergeMap` when computations are **dependent** (the second needs the first's result).
