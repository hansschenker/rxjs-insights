# Insight 02 — Observable as {Time, Value} Pairs

> Part of [rxjs-insight-groups.md](rxjs-insight-groups.md)  
> Primary sources: [rxjs-essential-knowledge.txt](rxjs-essential-knowledge.txt), [rxjs-insights-grok-4.txt](rxjs-insights-grok-4.txt), [rxjs-insights-gemini.txt](rxjs-insights-gemini.txt), [rxjs-fundamental-insights-chatgpt-45-version-2-refined.txt](rxjs-fundamental-insights-chatgpt-45-version-2-refined.txt)

---

## The Formal Model

An Observable is a **lazy, potentially infinite sequence of value–time pairs**:

```
Observable = [{T, a}…]
```

Where:
- `T` is the point in time at which the value was emitted
- `a` is the value emitted at that point

This is the most precise single-sentence definition of an Observable. It captures both what the stream carries (values) and when it carries them (time), treating both as equally fundamental.

No values are stored. The sequence is constructed on demand — lazily — and each pair is delivered to subscribers as it arrives, then discarded.

---

## Two Kinds of Streams

The `{T, a}` model immediately reveals two fundamentally different kinds of Observable:

### Continuous Streams

A **continuous stream** produces values that vary without discrete breaks. The `T` dimension is dense — values arrive at a regular or near-regular rate:

- A clock ticking every second: `{0s, 0}, {1s, 1}, {2s, 2}, …`
- An audio signal sampled at 44 100 Hz
- An animation frame counter at 60 fps
- A temperature sensor polled every 100 ms

For continuous streams, the *spacing* between emissions is meaningful. Time-based operators (`throttleTime`, `sampleTime`, `bufferTime`) are the natural tools.

### Discrete Event Streams

A **discrete event stream** produces values only at specific, unpredictable moments. The `T` dimension is sparse — long silences punctuated by single emissions:

- A click event: `{T_click, MouseEvent}`
- A GPS fix: `{T_fix, Coordinates}`
- A stock quote: `{T_tick, Price}`
- An HTTP response: `{T_response, ResponseBody}`

For discrete streams, the *value* dimension dominates. Value-based operators (`map`, `filter`, `take`) are the primary tools.

---

## The Two Operator Axes

The `{T, a}` model directly motivates the division of RxJS operators into two orthogonal categories:

### Value-Based Operators — act on `a`

These operators transform, filter, or accumulate the `a` without caring about the `T`:

| Operator | What it does to `a` |
|---|---|
| `map(fn)` | Replaces each `a` with `fn(a)` |
| `filter(pred)` | Drops values where `pred(a)` is false |
| `reduce(fn, seed)` | Folds all `a` into a single accumulated value |
| `scan(fn, seed)` | Like `reduce` but emits each intermediate `a` |
| `take(n)` | Keeps only the first `n` values |
| `skip(n)` | Drops the first `n` values |
| `distinct()` | Drops `a` values already seen |
| `distinctUntilChanged()` | Drops `a` if equal to the previous `a` |

### Time-Based Operators — act on `T`

These operators reshape *when* values are emitted, often ignoring or compressing the `a`:

| Operator | What it does to `T` |
|---|---|
| `debounceTime(ms)` | Emits `a` only if `T` is at least `ms` after the previous emission |
| `throttleTime(ms)` | Emits the first `a` in each `ms` window, suppresses the rest |
| `delay(ms)` | Shifts every `T` forward by `ms` |
| `timeout(ms)` | Errors if no `a` arrives within `ms` |
| `interval(ms)` | Generates `{0, 1, 2, …}` with `T` spacing of `ms` |
| `timer(delay, ms)` | Like `interval` but with an initial delay |
| `bufferTime(ms)` | Collects all `a` in a `ms` window into an array |
| `windowTime(ms)` | Emits a nested Observable for each `ms` window |
| `sampleTime(ms)` | Emits the most recent `a` every `ms` |
| `audit(fn)` | Like `throttleTime` but emits the *last* value in the window |

---

## Operators That Act on Both Axes

Some operators coordinate both `T` and `a` simultaneously:

- **`timestamp()`** — augments each `a` with its actual `T`, producing `{value: a, timestamp: T}`.
- **`timeInterval()`** — produces the time delta `T_n − T_{n−1}` between consecutive emissions.
- **`withLatestFrom(other$)`** — samples `a` from `other$` at the `T` of the source emission, joining both values.
- **`combineLatest([…])`** — emits the latest `a` from each source stream whenever any `T` fires.

---

## Finite vs Infinite Sequences

The `{T, a}` model also clarifies the distinction between finite and infinite Observables:

- **Finite**: the sequence has a last pair and then emits a `complete` signal. `from([1,2,3])`, `timer(1000)`, an HTTP request.
- **Infinite**: the sequence never emits `complete`. `interval(1000)`, `fromEvent(document, 'click')`. These run until explicitly unsubscribed or a limiting operator (`take`, `takeUntil`) terminates them.

This maps directly onto Haskell's finite vs lazy-infinite lists (see Insight 01).

---

## The Three Signals

Every `{T, a}` stream is bounded by exactly two terminal signals:

```
next(a)      — deliver a value pair {T, a}
error(e)     — terminate the stream with an error; no further pairs
complete()   — terminate the stream successfully; no further pairs
```

After either `error` or `complete`, the stream is closed. No further `{T, a}` pairs will arrive. This contract is enforced by the Observable specification.

---

## Why This Model Is Useful

### 1. It separates concerns cleanly

"I want to transform the values" → reach for value-based operators.  
"I want to control the timing" → reach for time-based operators.  
When a problem involves both, compose one of each.

### 2. It explains what Observable is *not*

An Observable is **not** an array with a time dimension added. An array holds all `a` values in memory simultaneously and has no `T` axis at all. An Observable holds none of its `a` values — each `{T, a}` pair is delivered once and gone. This is why operators like `toArray()` exist: they convert from the streaming model back into a batched one.

### 3. It motivates the signal-processing analogy

An Observable is a **digital signal**: a discrete sequence of samples taken at points in time. RxJS operators are **digital filters**:

| Signal processing | RxJS equivalent |
|---|---|
| Low-pass filter (remove high-freq noise) | `debounceTime()` |
| Sampling rate reduction | `throttleTime()` |
| Window function (batch processing) | `buffer()`, `window()` |
| Integration / accumulation | `scan()` |
| Differentiation (compare consecutive samples) | `pairwise()` |
| Signal mixing | `merge()`, `combineLatest()` |

This analogy is precise enough to be useful: both domains deal with sequences of `{T, value}` pairs and apply transformations that can be characterised by their frequency-domain behaviour.

### 4. It grounds the hot/cold distinction

A **cold Observable** starts its `T` clock from zero on each subscription — every subscriber gets a fresh sequence. A **hot Observable** has a `T` clock that runs independently; late subscribers join mid-sequence and miss earlier pairs (see Insight 11).
