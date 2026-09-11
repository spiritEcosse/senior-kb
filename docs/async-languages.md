# Async Across Languages: JavaScript, Rust, Go, C++

## The core problem

Every language solves the same problem: **a thread blocked on I/O is a wasted thread**. An OS thread costs ~1–8 MB of stack and a context switch costs ~1–5 µs, so "one thread per connection" collapses somewhere around 10k connections (the classic C10k problem).

The solutions differ in *who* does the scheduling and *what* the unit of suspension is:

| Language | Unit | Scheduler | Stack | Parallel? |
|---|---|---|---|---|
| Python | coroutine | `asyncio` event loop (single-threaded) | stackless (heap frames) | No (GIL) |
| JavaScript | Promise / async fn | runtime event loop (libuv / browser) | stackless | No (single thread + workers) |
| Go | goroutine | runtime M:N scheduler, work-stealing | growable stack (starts 2–8 KB) | Yes |
| Rust | `Future` | user-chosen executor (tokio, async-std) | stackless state machine | Yes (multi-thread runtime) |
| C++ | coroutine (C++20) | none built-in — you write it | stackless (heap frame) | Depends on your executor |

Two axes explain almost every difference:

1. **Stackful vs stackless.** Go gives each goroutine a real (growable) stack, so any function can block anywhere. Everyone else compiles `async` functions into state machines with no stack, so only explicitly marked functions can suspend.
2. **Batteries included vs bring-your-own.** Go and JS ship a runtime you cannot replace. Python ships one you *can* replace (uvloop). Rust ships a `Future` trait and no executor. C++ ships a coroutine *language feature* and essentially nothing else.

---

## Function colouring

"What colour is your function?" — in stackless languages, `async` infects call sites: an `async` function can only be awaited from another `async` function, so the annotation propagates up the whole call graph.

```python
# Python — coloured
async def fetch(url): ...
def handler():
    fetch(url)            # returns a coroutine object, does nothing
    await fetch(url)      # SyntaxError — handler is not async
```

```javascript
// JavaScript — coloured (top-level await in ESM is the escape hatch)
async function fetch_(url) { ... }
function handler() {
  fetch_(url);            // returns a Promise, floats away (unhandled rejection risk)
}
```

```rust
// Rust — coloured, and calling without .await does nothing at all
async fn fetch(url: &str) -> String { ... }
fn handler() {
    fetch(url);           // warning: unused implementer of `Future`; no work happens
}
```

```go
// Go — NOT coloured. Any function can block; the runtime parks the goroutine.
func fetch(url string) string { ... }   // ordinary blocking function
func handler() {
    go fetch(url)          // any call site can be made concurrent
}
```

Go is the outlier: because goroutines are stackful, blocking is a *runtime* concern, not a *type* concern. The cost is that you cannot tell from a signature whether a function blocks, and you lose `await` as an explicit yield point.

C++20 is coloured in a stricter way: a function is a coroutine if its body contains `co_await`/`co_yield`/`co_return`, which is a *body* property, not a signature property — the caller only sees the return type (`task<T>`).

---

## Lazy vs eager

This is the single most common source of cross-language bugs.

| Language | When does the work start? |
|---|---|
| JavaScript | **Eager** — the body runs up to the first `await` as soon as you call it |
| Python | **Lazy-ish** — nothing runs until awaited or wrapped in `create_task` |
| Rust | **Lazy** — nothing runs until polled (`.await` or `spawn`) |
| Go | **Eager** — `go f()` schedules immediately |
| C++20 | Depends on the `promise_type` (`initial_suspend` returns `suspend_always` → lazy, `suspend_never` → eager) |

```javascript
// JS: both requests are already in flight before the first await
const a = fetchUser();        // started
const b = fetchOrders();      // started
const [u, o] = [await a, await b];   // ~max(t1, t2)
```

```python
# Python: sequential — the second call isn't created until the first resolves
u = await fetch_user()        # ~t1
o = await fetch_orders()      # ~t2   → total t1 + t2

# Concurrent version needs an explicit scheduler call
u, o = await asyncio.gather(fetch_user(), fetch_orders())
```

```rust
// Rust: futures are inert values; join! polls them on one task
let (u, o) = tokio::join!(fetch_user(), fetch_orders());   // concurrent, same task
let h = tokio::spawn(fetch_user());                        // concurrent, may be another thread
```

!!! warning "The Rust trap"
    `let fut = fetch();` followed later by `fut.await` looks like it "started early" — it did not. A Rust `Future` makes no progress unless something polls it. Forgetting `.await` is a warning, not an error, and the work silently never happens.

---

## Fan-out: run N things concurrently

**Python**

```python
import asyncio, httpx

async def main(urls):
    async with httpx.AsyncClient() as client:
        results = await asyncio.gather(*(client.get(u) for u in urls))
    return [r.status_code for r in results]

# Python 3.11+: structured concurrency — cancels siblings on failure
async def main(urls):
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(fetch(u)) for u in urls]
    return [t.result() for t in tasks]
```

**JavaScript**

```javascript
const results = await Promise.all(urls.map(u => fetch(u)));       // fails fast
const settled = await Promise.allSettled(urls.map(u => fetch(u))); // never rejects
const first   = await Promise.race(urls.map(u => fetch(u)));       // first settled
const firstOk = await Promise.any(urls.map(u => fetch(u)));        // first fulfilled
```

| Combinator | Python | JavaScript | Rust (tokio) | Go |
|---|---|---|---|---|
| All, fail fast | `gather()` / `TaskGroup` | `Promise.all` | `try_join!` | `errgroup.Group` |
| All, collect errors | `gather(return_exceptions=True)` | `Promise.allSettled` | `join_all` | `sync.WaitGroup` |
| First to finish | `wait(FIRST_COMPLETED)` | `Promise.race` | `select!` | `select` on channels |
| First success | — | `Promise.any` | manual | manual |

**Go**

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) ([]int, error) {
    g, ctx := errgroup.WithContext(ctx)   // ctx cancelled when any goroutine errors
    codes := make([]int, len(urls))       // index-per-goroutine: no lock needed

    for i, u := range urls {
        i, u := i, u                      // pre-Go 1.22: rebind loop vars!
        g.Go(func() error {
            req, _ := http.NewRequestWithContext(ctx, "GET", u, nil)
            resp, err := http.DefaultClient.Do(req)
            if err != nil { return err }
            defer resp.Body.Close()
            codes[i] = resp.StatusCode
            return nil
        })
    }
    return codes, g.Wait()
}
```

**Rust**

```rust
use futures::future::try_join_all;

async fn fetch_all(urls: Vec<String>) -> Result<Vec<u16>, reqwest::Error> {
    let client = reqwest::Client::new();
    // Concurrent on one task — no Send bound needed, no thread hop
    try_join_all(urls.iter().map(|u| {
        let client = &client;
        async move { Ok(client.get(u).send().await?.status().as_u16()) }
    })).await
}

// Parallel across threads — each future must be Send + 'static
let handles: Vec<_> = urls.into_iter()
    .map(|u| tokio::spawn(async move { fetch(u).await }))
    .collect();
for h in handles {
    let r = h.await.expect("task panicked");   // JoinError wraps panics
}
```

**C++20 (with a library — here `cppcoro`-style)**

```cpp
#include <cppcoro/task.hpp>
#include <cppcoro/when_all.hpp>

cppcoro::task<int> fetch(std::string url);

cppcoro::task<std::vector<int>> fetch_all(std::vector<std::string> urls) {
    std::vector<cppcoro::task<int>> tasks;
    for (auto& u : urls) tasks.push_back(fetch(u));       // lazy: not started yet
    co_return co_await cppcoro::when_all(std::move(tasks));
}
```

C++26 adds `std::execution` (senders/receivers, ex-P2300), which finally gives the standard library a composition model. Until then, "async C++" means picking one of Asio, libunifex, cppcoro, folly::coro or stdexec — they do not interoperate.

---

## Cancellation

The deepest divergence between these languages.

| Language | Mechanism | Cancellation is… |
|---|---|---|
| Python | `task.cancel()` raises `CancelledError` at the next await point | Pre-emptive at await points |
| JavaScript | `AbortController` / `AbortSignal`, cooperative by convention | Advisory — a Promise cannot be cancelled |
| Rust | Drop the future | Pre-emptive: the state machine stops being polled |
| Go | `context.Context` + `ctx.Done()` channel | Cooperative — the callee must check |
| C++ | `std::stop_token` (C++20) | Cooperative |

```python
task = asyncio.create_task(long_job())
task.cancel()
try:
    await task
except asyncio.CancelledError:
    ...                      # cleanup; re-raise unless you own the cancellation

# Timeout = cancellation with a deadline
async with asyncio.timeout(5):     # 3.11+
    await long_job()
```

```javascript
const ctrl = new AbortController();
setTimeout(() => ctrl.abort(), 5000);
const res = await fetch(url, { signal: ctrl.signal });   // rejects with AbortError
// But: a plain `await sleep(10_000)` cannot be aborted — nothing to cancel.
```

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()                      // always defer cancel — leaked contexts leak goroutines

select {
case res := <-work(ctx):
    return res, nil
case <-ctx.Done():
    return nil, ctx.Err()           // context.DeadlineExceeded
}
```

```rust
// Cancellation = dropping the future. timeout() drops the inner future on expiry.
match tokio::time::timeout(Duration::from_secs(5), long_job()).await {
    Ok(v)  => v,
    Err(_) => return Err(Timeout),
}
// select! drops the losing branches
tokio::select! {
    v = long_job() => v,
    _ = shutdown.recv() => return,
}
```

!!! danger "Rust: cancellation safety"
    `tokio::select!` drops the futures of the branches that lose the race. If a future had already consumed bytes from a socket into a local buffer, those bytes are gone. A future is *cancel-safe* only if dropping it mid-poll loses no state — `AsyncReadExt::read` is, `AsyncBufReadExt::read_line` is not. Inside `select!`, prefer holding state outside the future or using `tokio::pin!` on a future you keep across iterations.

!!! danger "Go: goroutine leaks"
    Go cannot kill a goroutine from the outside. `go worker()` with no `ctx` and no exit condition is a permanent leak; a send on an unbuffered channel with no receiver parks that goroutine forever. Every goroutine needs an owner and a termination path. Detect with `goleak` in tests, `runtime.NumGoroutine()` and `/debug/pprof/goroutine` in production.

---

## Scheduling and parallelism

**Python** — one event loop per thread, one thread executing Python bytecode (GIL). A CPU-bound coroutine that never awaits blocks *everything*. Offload with `run_in_executor` / `asyncio.to_thread` (I/O, C extensions) or `ProcessPoolExecutor` (CPU). Swap in `uvloop` for ~2–4× loop throughput.

**JavaScript** — a single-threaded loop with a strict phase order and two queues:

```javascript
console.log('1');
setTimeout(() => console.log('2 macrotask'), 0);
Promise.resolve().then(() => console.log('3 microtask'));
queueMicrotask(() => console.log('4 microtask'));
process.nextTick(() => console.log('5 nextTick'));   // Node only, highest priority
console.log('6');
// 1, 6, 5, 3, 4, 2
```

Microtasks (promises) drain **completely** before the next macrotask (timers, I/O callbacks). An infinite microtask chain starves I/O permanently. CPU work goes to `worker_threads` (Node) or Web Workers (browser) — these are real threads with no shared heap (only `SharedArrayBuffer`/transferables).

**Go** — the G-M-P scheduler: **G** goroutines, **M** OS threads, **P** processors (`GOMAXPROCS`, default = number of cores). Each P holds a local run queue; idle Ps steal from busy ones. Since Go 1.14 the scheduler is *asynchronously pre-emptive* (signal-based), so a tight loop no longer wedges a P. Blocking syscalls detach the M from its P so other goroutines keep running. Goroutines start with ~2 KB of stack, copied and grown on demand, so millions are practical.

**Rust** — you choose. `#[tokio::main]` gives a multi-thread work-stealing runtime; `#[tokio::main(flavor = "current_thread")]` gives a single-threaded one. Tasks are **not** pre-empted: a task that computes without awaiting blocks its worker thread. Use `tokio::task::spawn_blocking` for blocking/CPU work (separate, larger pool up to 512 threads by default) and `tokio::task::yield_now()` in long compute loops. `spawn` requires `Send + 'static`; `spawn_local` (with a `LocalSet`) does not.

**C++** — there is no scheduler unless you supply one. `co_await` transfers control to whatever the awaiter's `await_suspend` decides; with Asio that's `io_context::run()` on however many threads you started.

```cpp
// Asio: a thread pool draining one io_context
asio::io_context io;
std::vector<std::thread> pool;
for (int i = 0; i < 4; ++i) pool.emplace_back([&]{ io.run(); });

asio::co_spawn(io, handle_connection(std::move(sock)), asio::detached);
```

---

## Synchronisation primitives

| Need | Python | JavaScript | Go | Rust (tokio) |
|---|---|---|---|---|
| Mutual exclusion | `asyncio.Lock` | — (single thread) | `sync.Mutex` | `tokio::sync::Mutex` |
| Limit concurrency | `asyncio.Semaphore` | `p-limit` | buffered channel | `tokio::sync::Semaphore` |
| Message passing | `asyncio.Queue` | `EventEmitter` / streams | channels (idiomatic) | `mpsc` / `broadcast` / `watch` |
| Wait for N | `gather` / `TaskGroup` | `Promise.all` | `sync.WaitGroup` | `JoinSet` |
| One-shot signal | `asyncio.Event` | Promise | `close(ch)` | `oneshot` |

```python
sem = asyncio.Semaphore(10)
async def bounded(url):
    async with sem:
        return await fetch(url)
```

```go
// Buffered channel as a semaphore
sem := make(chan struct{}, 10)
sem <- struct{}{}          // acquire
defer func() { <-sem }()   // release
```

```rust
let sem = Arc::new(Semaphore::new(10));
let permit = sem.clone().acquire_owned().await?;   // dropped = released
```

!!! note "Rust: `std::sync::Mutex` vs `tokio::sync::Mutex`"
    Use `std::sync::Mutex` for short critical sections with no `.await` inside — it's faster and its guard is not `Send` across awaits (the compiler enforces this). Use `tokio::sync::Mutex` only when you must hold the lock *across* an `.await`. Holding any lock across an await is a deadlock risk and serialises your tasks; restructuring to avoid it is usually the right fix.

!!! note "Go: channels vs mutex"
    "Share memory by communicating" is a guideline, not a law. A mutex around a map is simpler and faster than a goroutine owning the map behind a channel. Use channels for *ownership transfer* and *signalling*; use `sync.Mutex`/`sync.RWMutex`/`atomic` for shared state.

---

## Error propagation

```python
# Exceptions travel through await naturally
try:
    await fetch(url)
except httpx.HTTPError as e:
    ...
# TaskGroup raises ExceptionGroup (3.11+)
try:
    async with asyncio.TaskGroup() as tg: ...
except* ValueError as eg:
    for e in eg.exceptions: ...
```

```javascript
// Unhandled rejection kills the Node process by default since v15
try { await fetch(url); } catch (e) { ... }
// Fire-and-forget MUST be caught:
doWork().catch(err => logger.error(err));
```

```go
// Errors are values; no exceptions. A goroutine cannot return an error to its parent —
// use a channel or errgroup. An unrecovered panic in ANY goroutine kills the process.
go func() {
    defer func() { if r := recover(); r != nil { log.Println("recovered:", r) } }()
    work()
}()
```

```rust
// Result<T, E> + `?` through async fns; panics in a spawned task become JoinError
match handle.await {
    Ok(Ok(v))  => v,
    Ok(Err(e)) => return Err(e),      // the task's own error
    Err(join)  => if join.is_panic() { /* task panicked */ },
}
```

| | Failure of one task kills siblings? |
|---|---|
| Python `gather()` | No (siblings keep running; first exception propagates) |
| Python `TaskGroup` | Yes — cancels the group |
| JS `Promise.all` | No — rejects immediately, siblings keep running |
| Go `errgroup.WithContext` | Yes, cooperatively (ctx cancelled) |
| Rust `try_join!` | Yes — losing futures are dropped |
| Rust `JoinSet` | No, until you `abort_all()` |

---

## Async I/O under the hood

All of these end at the same OS primitives:

| OS | Readiness | Completion |
|---|---|---|
| Linux | `epoll` | `io_uring` |
| BSD/macOS | `kqueue` | — |
| Windows | — | IOCP |

- **Python**: `selectors` → `epoll`/`kqueue`; `uvloop` wraps libuv.
- **Node**: libuv → `epoll`/`kqueue`/IOCP, plus a 4-thread pool (`UV_THREADPOOL_SIZE`) for file I/O and DNS, which *are not* natively async on Linux.
- **Go**: `netpoller` (`epoll`/`kqueue`) integrated into the scheduler — blocking-looking calls park the goroutine.
- **Rust**: mio → `epoll`/`kqueue`/IOCP; `tokio-uring` / `glommio` for `io_uring`.
- **C++**: Asio wraps the same, and can use `io_uring` on recent Linux.

Readiness ("the socket is now readable, go read it") vs completion ("your read finished, here's the buffer") is why Windows and `io_uring` sometimes need a different buffer ownership model — `io_uring` wants the buffer to stay alive for the whole operation, which conflicts with Rust's "cancel = drop" model and is why `tokio-uring` uses owned buffers.

---

## Performance ballpark

Order-of-magnitude figures; measure your own workload.

| | Spawn cost | Memory per unit | Practical count |
|---|---|---|---|
| OS thread | ~10–100 µs | 1–8 MB (virtual) | thousands |
| Go goroutine | ~0.3 µs | ~2 KB → grows | millions |
| Rust task (tokio) | ~0.1–1 µs | ~64–300 B + future size | millions |
| Python coroutine | ~1 µs | ~0.5–1 KB | ~100k |
| JS promise | ~0.1 µs | ~100 B | millions |

Throughput for a trivial HTTP echo server usually orders as: Rust ≈ C++ > Go > Node > Python (CPython+uvloop), with Rust/C++ typically 3–10× CPython. But at this level the bottleneck is almost always the database, the serialiser, or the network — not the runtime.

---

## Common pitfalls by language

**Python**
- Calling a blocking library (`requests`, `psycopg2` sync, `time.sleep`) inside a coroutine — freezes the whole loop. Use `asyncio.to_thread` or an async driver.
- `create_task` without keeping a reference: the loop only holds a weak reference and the task can be garbage-collected mid-flight. Keep a set of strong refs.
- Swallowing `CancelledError` — it must propagate (it inherits from `BaseException` since 3.8 for exactly this reason).

**JavaScript**
- `array.forEach(async x => ...)` — `forEach` ignores the returned promises; nothing is awaited. Use `for...of` with `await`, or `Promise.all(array.map(...))`.
- `await` inside a loop when the iterations are independent — serialises what should be concurrent.
- Unhandled rejections from fire-and-forget promises.
- A synchronous CPU loop blocks every request on the process.

**Go**
- Goroutine leaks (see above).
- Pre-Go 1.22, loop variables were shared across iterations — `for i, v := range` captured by a closure gave the last value. Go 1.22 made them per-iteration; older code still needs `i := i`.
- Sending on a closed channel panics; closing twice panics. The *sender* closes, never the receiver.
- `WaitGroup.Add` must happen before `go`, not inside the goroutine.

**Rust**
- Forgetting `.await` (nothing runs).
- Blocking inside an async task (`std::fs`, `std::thread::sleep`, heavy CPU) starves the worker thread — use `spawn_blocking`.
- Holding a `std::sync::MutexGuard` across `.await` — compile error under `Send` bounds, and a deadlock design smell regardless.
- Cancellation-safety bugs in `select!`.
- `async fn` in traits: stabilised for static dispatch in Rust 1.75; `dyn` still needs `async-trait`.

**C++**
- A coroutine's frame is heap-allocated and lives until it completes or is destroyed. Capturing a reference to a local in the *caller* is a dangling-reference bug — coroutine parameters are copied into the frame, but lambda captures are not.
- `co_await`ing a temporary whose lifetime ends at the end of the full expression.
- Mixing two coroutine libraries in one program.

---

## Choosing

| Situation | Pick |
|---|---|
| 10k+ idle connections, mostly I/O | Go, Rust, Node |
| Predictable tail latency, no GC pauses | Rust, C++ |
| Team velocity on a network service | Go |
| Existing Python/ML stack | Python + `asyncio` (+ uvloop) |
| Browser or shared JS/TS codebase | JavaScript |
| Embedding into an existing C++ codebase | C++20 coroutines + Asio |
| CPU-bound work | Any language with real threads — or a process pool in Python/Node |

---

## Interview questions

**Why is Go's `go f()` cheaper than a thread?**
Goroutines are multiplexed onto a small number of OS threads (M:N) by the runtime, start with ~2 KB of growable stack instead of a fixed MB-sized one, and switch in user space (~100 ns) without a kernel trap.

**What does `async` actually compile to in Rust/C++?**
A state machine: the compiler splits the function at each suspension point into states, and the locals that live across a suspension become fields of a generated struct. Rust's is `!Unpin` and must be pinned before polling; C++'s frame is heap-allocated (the allocation may be elided).

**Why does Python's `asyncio` not give parallelism?**
One event loop runs on one thread, and the GIL allows only one thread to execute bytecode. Concurrency comes from overlapping *waits*, not from overlapping computation.

**What's the difference between `Promise.all` and `Promise.allSettled`?**
`all` rejects on the first rejection (the other promises keep running, uncancelled); `allSettled` always fulfils with an array of `{status, value|reason}`.

**Rust: what is cancellation safety and when does it matter?**
A future is cancel-safe if dropping it mid-poll loses no data. It matters inside `select!` and `timeout`, which drop the futures that don't win.

**Go: how do you stop a goroutine?**
You don't — you ask it to stop. Pass a `context.Context` and check `ctx.Done()`, or close a quit channel. There is no `kill`.

**Why does Node have both a microtask and a macrotask queue?**
So that promise continuations run before the loop moves to the next I/O phase, giving promises "run as soon as possible" semantics. The cost is that an unbounded microtask chain starves I/O.

---
