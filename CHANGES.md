# Changes

Summary of all work done in this repository from initial creation to current state.

---

## Session — 2026-06-13

### 1. Repository bootstrap

- Initialised git repository (`git init`).
- Created **`CLAUDE.md`** — a guidance file for Claude Code describing what the directory is, listing all source `.txt` files with descriptions, and documenting working conventions.
- Created **`rxjs-insight-groups.md`** — a cross-file thematic index that groups insights from all 11 source `.txt` files into 14 named clusters, each with a short summary and a list of the source files where the theme appears.

### 2. Insight group analysis documents

Created one detailed analysis document per insight group (`rxjs-insight-01.md` through `rxjs-insight-14.md`):

| File | Insight group |
|---|---|
| `rxjs-insight-01.md` | Historical Lineage: Haskell → LINQ → Rx.NET → RxJS → TypeScript |
| `rxjs-insight-02.md` | Observable as `{Time, Value}` pairs; continuous vs discrete; value-based vs time-based operators |
| `rxjs-insight-03.md` | Observer / Iterator Duality: push vs pull, inverted interfaces, cold = iterator-like, hot = observer-like |
| `rxjs-insight-04.md` | Observable as Functor / Applicative / Monad; four `flatMap` variants; monad laws; Kleisli composition |
| `rxjs-insight-05.md` | Reactive Programming Paradigm; dependency graph / DAG; Reactive Manifesto four properties |
| `rxjs-insight-06.md` | Functional Reactive Programming; pure functions, immutability, Behaviours vs Events, side effects at edges |
| `rxjs-insight-07.md` | RxJS as a DSL; operators as vocabulary; `pipe()` as grammar; domain independence; signal processing analogy |
| `rxjs-insight-08.md` | Unifying Async Model; five pre-RxJS patterns unified behind one `Observable` interface |
| `rxjs-insight-09.md` | Operator Taxonomy; three axes (role / order / aspect); operator family trees; 16 essential operators |
| `rxjs-insight-10.md` | Three-Step Workflow: Create → Pipe → Subscribe; laziness; subscription tree activation and teardown |
| `rxjs-insight-11.md` | Hot vs Cold; unicast vs multicast; Subject variants; `share` / `shareReplay`; practical decision guide |
| `rxjs-insight-12.md` | Declarative vs Imperative; Observable does not store data; push-based delivery; statements vs expressions |
| `rxjs-insight-13.md` | Practical Concerns: subscription cleanup, error handling, backpressure, marble testing with `TestScheduler` |
| `rxjs-insight-14.md` | TypeScript Integration; `Observable<T>`, `OperatorFunction<T,R>`; type inference through `pipe()` |

### 3. Cross-linking

- Added a **Detail** column to the theme table in `CLAUDE.md` linking each row to its `rxjs-insight-xx.md` file.
- Added `— [detail](rxjs-insight-xx.md)` links to each group heading in `rxjs-insight-groups.md`.
- Expanded the **File Overview** section in `CLAUDE.md` with a second sub-table listing all 15 derived `.md` files with descriptions and links.

### 4. GitHub

- Created public GitHub repository: **https://github.com/hansschenker/rxjs-insights**
- Added **`README.md`** with a navigation table linking to all 14 insight documents and a source notes table.
- Added the GitHub URL to both `CLAUDE.md` and `README.md`.

---

## File inventory

| File | Type | Description |
|---|---|---|
| `CLAUDE.md` | Guide | Claude Code guidance; file overview; insight group table with links |
| `README.md` | Guide | Public-facing introduction and navigation |
| `CHANGES.md` | Guide | This file |
| `rxjs-insight-groups.md` | Index | All 14 groups with summaries, source references, and detail links |
| `rxjs-insight-01.md` … `rxjs-insight-14.md` | Analysis | Deep-dive documents, one per insight group |
| `rxjs-insights-list.txt` | Source | Condensed insight list |
| `Rxjs-insights-summariy-0-evolution.txt` | Source | Haskell → LINQ → Rx.NET → RxJS narrative |
| `rxjs-insights-claude-01.txt` / `rxjs-insights-claude-all.txt` | Source | Claude-generated deep-dives |
| `rxjs-insights-grok-4.txt` / `rxjs-insights-grok-4-notebooklm.txt` | Source | Grok 4 content |
| `rxjs-essential-characteristics-grok3-com.txt` | Source | Grok 3 characteristics overview |
| `rxjs-essential-knowledge.txt` | Source | Core knowledge reference |
| `rxjs-essentials-vs-javascript-essentials.txt` | Source | RxJS vs vanilla JS contrast |
| `rxjs-essentials-yakov-fain.txt` | Source | Yakov Fain book notes |
| `rxjs-essentials.txt` | Source | Observable-as-inverted-iterator essay |
| `rxjs-fundamental-insights-chatgpt-45-version-2-refined.txt` | Source | ChatGPT-refined fundamentals |
| `rxjs-insights-gemini.txt` | Source | Gemini-generated insights |
