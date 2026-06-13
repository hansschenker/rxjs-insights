# Insight 11 — Hot vs Cold Observables

> Part of [rxjs-insight-groups.md](rxjs-insight-groups.md)  
> Primary sources: [rxjs-insights-list.txt](rxjs-insights-list.txt), [rxjs-insights-grok-4.txt](rxjs-insights-grok-4.txt), [rxjs-insights-grok-4-notebooklm.txt](rxjs-insights-grok-4-notebooklm.txt)

---

## The Core Distinction

Every Observable falls into one of two categories based on its relationship to its subscribers:

| | Cold Observable | Hot Observable |
|---|---|---|
| **Execution** | Starts on subscribe | Runs independently of subscribers |
| **Subscribers** | Each gets its own execution | All share the same execution |
| **Missed values** | None — subscriber sees everything | Late subscribers miss past values |
| **Analogy** | Recorded movie | Live television broadcast |
| **Common examples** | `of`, `from`, `ajax`, HTTP requests | `Subject`, `fromEvent`, WebSocket |

---

## Cold Observables

A cold Observable contains its **data source inside** the Observable factory function. Every new subscriber triggers a completely fresh execution:

```typescript
// Cold: the timer starts fresh for each subscriber
const cold$ = new Observable<number>(subscriber => {
  console.log('New execution started');
  let i = 0;
  const id = setInterval(() => subscriber.next(i++), 1000);
  return () => clearInterval(id); // teardown
});

cold$.subscribe(x => console.log('A:', x)); // starts its own interval
// → "New execution started"
// → A: 0, A: 1, A: 2, …

setTimeout(() => {
  cold$.subscribe(x => console.log('B:', x)); // starts ANOTHER interval
  // → "New execution started"
  // → B: 0, B: 1, B: 2, …  (B starts from 0, not from where A is)
}, 3000);
```

The two subscribers are completely independent. B gets its own sequence from zero — it does not share A's execution.

### Coldness Is the Default

All RxJS creation operators produce cold Observables:

```typescript
const numbers$ = of(1, 2, 3);
numbers$.subscribe(console.log); // 1, 2, 3
numbers$.subscribe(console.log); // 1, 2, 3 again (new execution)

const request$ = ajax.getJSON('/api/data');
request$.subscribe(handler1); // fires HTTP request
request$.subscribe(handler2); // fires ANOTHER HTTP request
```

The second `ajax` subscription is a common footgun: developers expect one HTTP request but get two.

---

## Hot Observables

A hot Observable has its **data source outside** the Observable — it is already running. Subscribers tap into a stream that exists independently of them:

```typescript
// Subject is the simplest hot Observable
const hot$ = new Subject<number>();

hot$.next(1); // emitted before any subscriber — nobody receives this

hot$.subscribe(x => console.log('A:', x));
hot$.next(2); // A: 2

hot$.subscribe(x => console.log('B:', x));
hot$.next(3); // A: 3, B: 3  (both receive it)
hot$.next(4); // A: 4, B: 4
```

Subscriber A missed the value `1` because it subscribed late. Subscriber B missed `1` and `2`.

### DOM Events Are Naturally Hot

```typescript
const clicks$ = fromEvent(document, 'click');
// The event source exists independently of RxJS

clicks$.subscribe(handler1); // both share the same DOM event listener
clicks$.subscribe(handler2); // (under the hood, RxJS may add separate listeners,
                              //  but conceptually the source is shared)
```

---

## Bridging Cold to Hot: Multicasting

The need to **share a cold Observable's execution among multiple subscribers** is the motivation for multicasting operators. They convert cold → hot.

### `share()`

Multicasts to current subscribers; when the last unsubscribes the source stops; re-subscribing restarts it:

```typescript
const shared$ = ajax.getJSON('/api/data').pipe(share());

shared$.subscribe(handler1); // fires ONE HTTP request
shared$.subscribe(handler2); // shares the same request
// handler1 and handler2 both receive the same response
```

### `shareReplay(n)`

Like `share()` but **caches** the last `n` emissions. Late subscribers receive the cached values immediately, without triggering a new source execution:

```typescript
const cached$ = ajax.getJSON('/api/config').pipe(shareReplay(1));

cached$.subscribe(handler1); // fires HTTP request, caches response
// ... later
cached$.subscribe(handler2); // receives cached response immediately, no new request
```

`shareReplay(1)` is the standard pattern for:
- HTTP requests that should fire once but serve multiple subscribers.
- State streams that a late subscriber needs the current value of.
- Expensive computations shared across a component tree.

### `publish()` + `connect()`

Manual control over when multicasting begins:

```typescript
const source$ = interval(1000).pipe(publish()) as ConnectableObservable<number>;

source$.subscribe(x => console.log('A:', x));
source$.subscribe(x => console.log('B:', x));

// Neither A nor B receives anything yet — source hasn't started

source$.connect(); // now both receive values from the same source
```

---

## Subject: The Canonical Hot Observable

`Subject` is both an Observable and an Observer — you can both subscribe to it and push values into it:

```typescript
const subject$ = new Subject<number>();

// Observable side — subscribe
subject$.subscribe(x => console.log('A:', x));
subject$.subscribe(x => console.log('B:', x));

// Observer side — push values
subject$.next(1); // A: 1, B: 1
subject$.next(2); // A: 2, B: 2
subject$.complete();
```

Subject variants:

| Variant | Behaviour for late subscribers |
|---|---|
| `Subject<T>` | Miss all past values |
| `BehaviorSubject<T>(initial)` | Receive the current value immediately on subscribe |
| `ReplaySubject<T>(n)` | Receive the last `n` values immediately on subscribe |
| `AsyncSubject<T>` | Receive only the last value, after the subject completes |

`BehaviorSubject` is particularly important for state management — it always has a current value, so every subscriber immediately gets the latest state:

```typescript
const state$ = new BehaviorSubject<AppState>(initialState);

state$.subscribe(renderUI); // immediately renders with initialState
// ... later
dispatch(action);           // updates BehaviorSubject
state$.subscribe(logState); // immediately receives current state, not initial
```

---

## The Late Subscriber Problem

The fundamental tension of hot Observables: a subscriber that arrives after some emissions has **missed data with no way to recover it** (unless the source replays).

This is not always a problem — with `fromEvent(button, 'click')` you naturally expect to miss clicks that happened before subscribing. But with a data stream that emits a configuration object once on startup, a late subscriber would miss that configuration and operate with no data.

The solution is always one of:
1. Use `shareReplay(1)` — cache and replay the last value.
2. Use `BehaviorSubject` — always holds the current value for immediate delivery.
3. Use `ReplaySubject(n)` — hold the last `n` values for replay.

---

## Practical Decision Guide

| Scenario | Choice |
|---|---|
| HTTP request used in one place | Cold (default `ajax`) |
| HTTP request shared by multiple subscribers | `ajax.pipe(shareReplay(1))` |
| Application state | `BehaviorSubject` |
| Event bus / message broker | `Subject` |
| DOM events | Hot by nature (`fromEvent`) |
| Expensive computation shared across tree | Cold source `+ shareReplay(1)` |
| Stream that should replay on reconnect | `ReplaySubject(n)` |
