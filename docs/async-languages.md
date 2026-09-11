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

The **P2300** proposal (`std::execution`, senders/receivers) finally gives the standard library a composition model — see [the section below](#c-the-p2300-proposal-stdexecution). Until implementations catch up, "async C++" means picking one of Asio, libunifex, cppcoro, folly::coro or stdexec — they do not interoperate.

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

**C++** — there is no scheduler unless you supply one (P2300 standardises the *concept* of one, and P2079 adds a shared `parallel_scheduler`). `co_await` transfers control to whatever the awaiter's `await_suspend` decides; with Asio that's `io_context::run()` on however many threads you started.

```cpp
// Asio: a thread pool draining one io_context
asio::io_context io;
std::vector<std::thread> pool;
for (int i = 0; i < 4; ++i) pool.emplace_back([&]{ io.run(); });

asio::co_spawn(io, handle_connection(std::move(sock)), asio::detached);
```

---

## C++: the P2300 proposal (`std::execution`)

### What P2300 is

**P2300 — "`std::execution`" — is a WG21 proposal paper**, not a library someone shipped and then standardised. It is the document that *specifies* an asynchronous programming model for the C++ standard library: the vocabulary types, the concepts they must satisfy, the algorithms, and the rules for customising them. It was adopted into the C++26 working draft (as revision R10).

The paper's authors are Eric Niebler, Kirk Shoop, Lewis Baker, Michał Dominiak, Georgy Evtushenko, Lucian Radu Teodorescu, Lee Howes, Michael Garland and Bryce Adelstein Lelbach — a large share of them at **NVIDIA**, which is why the reference implementation lives at [NVIDIA/stdexec](https://github.com/NVIDIA/stdexec) and why the design is built to reach a GPU, not just a thread pool.

The lineage: P0443 (the old "executors" proposal, ~5 years, never landed) → libunifex (Meta's experiment with lazy senders) → P2300. The earlier proposals tried to standardise *where work runs*; P2300 standardises *how asynchronous work is described and composed*, and makes "where" one property of that description.

!!! note "Why a proposal matters here"
    Everything in this section is a specification that implementations must match, so the same pipeline can run on `stdexec`, a vendor's standard library, an embedded executor, or a GPU backend. That is the whole point: before P2300, "async C++" meant picking Asio *or* libunifex *or* cppcoro *or* folly::coro, and code written against one could not be composed with another.

### The four concepts

| Concept | Role | Analogy |
|---|---|---|
| **scheduler** | a handle to an execution context; `schedule(sch)` returns a sender that completes *on* it | tokio `Runtime`, Go's `P` |
| **sender** | a lazy *description* of async work, not the work itself | Rust `Future` |
| **receiver** | the continuation — three channels: `set_value`, `set_error`, `set_stopped` | Rust `Waker` + the `Poll` result |
| **operation_state** | what `connect(sender, receiver)` returns; the actual suspended state. Runs only when `start()`ed | Rust's pinned state machine |

The three completion channels are the design's sharpest idea. A sender does not just "resolve or reject" — it completes in exactly one of three ways, and the set of possible completions is **part of the type**, computed at compile time as `completion_signatures`:

```cpp
// Reading roughly as: this sender either yields an int, fails with exception_ptr,
// or reports that it was stopped.
using sigs = completion_signatures<
    set_value_t(int),
    set_error_t(std::exception_ptr),
    set_stopped_t()>;
```

Rust's `Future` has one output type and expresses cancellation by *not existing any more*; JavaScript has resolve/reject and no cancellation channel at all. `set_stopped` gives C++ a first-class "this was cancelled, and that is not an error" signal that composes through every algorithm.

### A pipeline

```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>

namespace ex = stdexec;

exec::static_thread_pool pool{8};
ex::scheduler auto sch = pool.get_scheduler();

ex::sender auto work =
      ex::schedule(sch)                         // start on the pool
    | ex::then([] { return load_rows(); })      // value channel: T -> U
    | ex::let_value([](std::vector<Row>& rows) { // rows live in the operation state
          return ex::just()
               | ex::bulk(rows.size(), [&rows](std::size_t i) { transform(rows[i]); })
               | ex::then([&rows] { return summarise(rows); });
      })
    | ex::upon_error([](std::exception_ptr) { return Summary::empty(); });

auto [summary] = ex::sync_wait(std::move(work)).value();   // connect + start + block
```

Nothing above runs until `sync_wait` connects the sender to a receiver and starts the resulting operation state. Build the pipeline, throw it away, and no work happened — the same lazy contract as a Rust `Future`, and the opposite of a JavaScript Promise.

Core algorithms:

| Kind | Algorithms |
|---|---|
| Sources | `just`, `just_error`, `just_stopped`, `schedule`, `read_env` |
| Value transforms | `then`, `let_value`, `bulk` |
| Error / stop handling | `upon_error`, `upon_stopped`, `let_error`, `let_stopped`, `stopped_as_optional` |
| Composition | `when_all`, `into_variant`, `split`, `starts_on`, `continues_on` |
| Consumers | `sync_wait`, `start_detached` |

`continues_on(sch)` (named `transfer` in earlier revisions) is the "hop to another context" operator — everything downstream of it completes on `sch`.

### Structured concurrency without allocation

This is what distinguishes P2300 from every other model on this page.

`connect()` returns the operation state **by value**. The caller owns it, it is neither copyable nor movable, and it must outlive the operation — so the natural place for it is the enclosing operation state, which is itself a member of *its* parent. A whole pipeline collapses into one nested object whose lifetime is a scope, and the "no heap allocation required" property falls out of that.

Compare:

- **Rust**: `tokio::spawn` boxes the future and requires `Send + 'static`. `join!` avoids the box but the task still lives in the runtime's slab.
- **C++20 coroutines alone**: each coroutine frame is a separate heap allocation (elidable in principle, rarely in practice across a library boundary).
- **Go**: every goroutine gets a real stack, growable but never free.
- **P2300**: one composite object, stack-allocatable, with the compiler able to see through the whole chain and inline it.

The cost is that lifetimes become your problem: a sender that borrows must not outlive what it borrows, and `start_detached` (fire-and-forget) deliberately breaks the structure — which is why `async_scope` exists to give detached work an owner.

### Cancellation

Cancellation travels through the receiver's **environment** — a bag of queries attached to the receiver, reachable as `get_env(rcvr)`:

```cpp
auto token = ex::get_stop_token(ex::get_env(rcvr));
if (token.stop_requested()) { ex::set_stopped(std::move(rcvr)); return; }
token.stop_callback_for_t<Fn> cb{token, Fn{...}};   // wake the I/O, cancel the kernel
```

Algorithms forward the token down the chain automatically, so `when_all` cancelling its siblings, or a timeout stopping a whole subtree, is the same mechanism rather than three different ones. Completion then arrives on `set_stopped`, distinct from an error.

| | Cancellation signal | Distinct from error? |
|---|---|---|
| Go | `ctx.Done()` channel, checked by hand | No — `ctx.Err()` is an error value |
| Rust | drop the future | No signal at all |
| Python | `CancelledError` raised at an await point | Yes (`BaseException`) |
| JS | `AbortSignal` → `AbortError` rejection | No |
| P2300 | stop token in, `set_stopped` out | **Yes, a separate channel** |

### Coroutine interop

P2300 does not replace C++20 coroutines — it is the layer they were missing. A sender is awaitable inside a coroutine, and a coroutine task is itself a sender:

```cpp
exec::task<int> handle(ex::scheduler auto sch) {
    co_await ex::schedule(sch);                          // resume on the pool
    auto [a, b] = co_await ex::when_all(fetch(1), fetch(2));
    co_return a + b;
}

auto [n] = ex::sync_wait(handle(sch)).value();           // the task IS a sender
```

Use coroutines where sequencing reads better as straight-line code; use sender algorithms where the structure is a graph (fan-out, error routing, retries) or where you cannot afford the frame allocation.

### The NVIDIA angle: the same pipeline on a GPU

`nvexec` (in the same repository) supplies `nvexec::stream_scheduler` and `multi_gpu_stream_scheduler`, backed by CUDA streams and built with `nvc++ -stdpar=gpu`. Because "where it runs" is just the scheduler you pass, the pipeline body does not change:

```cpp
nvexec::stream_context stream;
auto gpu = stream.get_scheduler();

auto work = ex::just(std::move(data))
          | ex::continues_on(gpu)                         // hop to the device
          | ex::bulk(n, [](std::size_t i, auto&& v) { v[i] = f(v[i]); })  // → CUDA kernel
          | ex::continues_on(cpu)                         // hop back
          | ex::then([](auto&& v) { return reduce(v); });
```

`bulk` on a stream scheduler becomes a kernel launch; the sender chain becomes work enqueued on the stream, and the stop token becomes stream cancellation. This is the reason NVIDIA drove the proposal: one composition model spanning a thread pool, an `io_uring` context and a GPU, instead of CUDA-specific plumbing glued to whatever the host code happened to use.

### Customisation, and what changed along the way

Early revisions customised algorithms with `tag_invoke` (P1895) — free functions found by ADL on a tag type. That was replaced by **member-function customisation points** (P2855) after complaints about compile-time scalability, and algorithm customisation now goes through *domains* attached to a sender's environment rather than by overloading globally.

A standard library that ships `std::execution` but no way to run anything in parallel would be useless, so **P2079** adds a shared `parallel_scheduler` (`get_parallel_scheduler()`, formerly `system_scheduler`/`system_context`) — a standard parallel execution context so you don't have to bring a thread pool just to say hello.

### Caveats

- **Complexity.** The model is deep template metaprogramming; error messages and compile times are the standing criticism, and the proposal's size and process drew public pushback during standardisation.
- **Availability.** Use `NVIDIA/stdexec` today (GCC 12+, Clang 16+, MSVC 14.43+, `nvc++` 25.9+ for GPU). Vendor standard-library implementations of `<execution>` lag the paper.
- **Naming churn.** `transfer` → `continues_on`, `system_scheduler` → `parallel_scheduler`, `tag_invoke` gone. Older blog posts and talks will not compile.
- **It is a model, not a runtime.** P2300 does not give you an HTTP client, a socket, or a timer wheel. It gives you the contract those things should expose.

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
- Building a sender pipeline and never `connect`ing/starting it — the code compiles and does nothing, exactly like a forgotten `.await` in Rust.
- Letting an operation state be destroyed while the operation is in flight, or letting a sender outlive what it borrows.
- `sync_wait` called from a thread of the same pool the pipeline runs on — the thread blocks waiting on work it was supposed to execute.
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
| Async C++ that must also target GPUs/accelerators | `std::execution` (P2300) + `stdexec`/`nvexec` |
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

**What problem does P2300 solve that C++20 coroutines don't?**
Coroutines gave C++ the *syntax* for suspension but no vocabulary types, no algorithms and no scheduler concept, so every library invented its own and none composed. P2300 specifies the model — scheduler/sender/receiver/operation_state, three completion channels, and a set of composable algorithms — so a pipeline can move between a thread pool, an `io_uring` context and a GPU by swapping the scheduler.

**Why does a sender complete on three channels instead of two?**
`set_value`, `set_error` and `set_stopped` separate "cancelled" from "failed". Go models cancellation as an error value, Rust as the absence of a future, JS as a rejection; making it a channel lets every algorithm forward and handle it uniformly.

**Why does Node have both a microtask and a macrotask queue?**
So that promise continuations run before the loop moves to the next I/O phase, giving promises "run as soon as possible" semantics. The cost is that an unbounded microtask chain starves I/O.

---
