# Threading vs asyncio vs Multiprocessing

## The core problem

Python has three concurrency models. Picking the wrong one costs performance; picking the right one is a 10x difference.

The root cause of the distinction is the **GIL**: only one thread executes Python bytecode at a time. This makes `threading` useless for CPU-bound work but fine for I/O-bound work (the GIL is released during I/O syscalls).

---

## Quick decision guide

| Workload | Use |
|---|---|
| Many network requests / DB queries | `asyncio` |
| Blocking I/O you can't make async | `threading` |
| CPU-heavy computation (ML, image processing) | `multiprocessing` |
| Mix of CPU + I/O | `multiprocessing` + `asyncio` inside each worker |

---

## threading

A thread is an OS-level execution unit sharing the same memory space. The GIL means only one thread runs Python bytecode at a time, but threads genuinely run in parallel during I/O waits.

**When to use:** blocking I/O calls you can't `await` (legacy libraries, `subprocess`, `time.sleep` in tests).

```python
import threading
import requests

results = {}

def fetch(url):
    results[url] = requests.get(url).status_code  # blocking — GIL released during network wait

urls = ["https://httpbin.org/get"] * 5
threads = [threading.Thread(target=fetch, args=(url,)) for url in urls]
for t in threads: t.start()
for t in threads: t.join()
print(results)
```

**ThreadPoolExecutor (preferred):**

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import requests

urls = ["https://httpbin.org/get"] * 10

with ThreadPoolExecutor(max_workers=5) as pool:
    futures = {pool.submit(requests.get, url): url for url in urls}
    for future in as_completed(futures):
        url = futures[future]
        print(url, future.result().status_code)
```

**Shared state requires locking:**

```python
import threading

counter = 0
lock = threading.Lock()

def increment():
    global counter
    with lock:
        counter += 1  # read-modify-write is not atomic

threads = [threading.Thread(target=increment) for _ in range(1000)]
for t in threads: t.start()
for t in threads: t.join()
print(counter)  # 1000 ✓ (without lock: unpredictable)
```

**Pitfalls:**
- Race conditions on shared mutable state
- Deadlocks when multiple locks are acquired in different order
- Memory overhead: ~8 MB stack per thread by default on Linux

---

## asyncio

Single-threaded cooperative multitasking. A coroutine voluntarily yields control at every `await`, allowing the event loop to run another coroutine. No GIL contention, no OS context switches — extremely cheap.

**When to use:** high-concurrency I/O (thousands of HTTP calls, WebSocket connections, DB queries with an async driver).

```python
import asyncio
import aiohttp

async def fetch(session, url):
    async with session.get(url) as resp:
        return resp.status

async def main():
    urls = ["https://httpbin.org/get"] * 100

    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        results = await asyncio.gather(*tasks)

    print(results)

asyncio.run(main())
```

**Event loop: what actually happens**

The event loop is a single infinite loop that does three things repeatedly:

1. **Check the ready queue** — run all callbacks/coroutines that are ready right now (no I/O wait)
2. **Poll I/O** — ask the OS (via `select`/`epoll`/`kqueue`) which file descriptors are ready; resume waiting coroutines
3. **Check scheduled callbacks** — run anything whose `asyncio.sleep()` timer has expired

```
┌─────────────────────────────────────────────────────┐
│                    Event Loop                        │
│                                                      │
│  ┌──────────────┐    ┌─────────────┐   ┌─────────┐  │
│  │  Ready Queue │ →  │  I/O Poll   │ → │ Timers  │  │
│  │  (run now)   │    │  (epoll)    │   │ (sleep) │  │
│  └──────────────┘    └─────────────┘   └─────────┘  │
│         ↑                   │                │       │
│         └───────────────────┴────────────────┘       │
│              completed I/O / expired timers           │
└─────────────────────────────────────────────────────┘
```

**Step by step — three coroutines, one thread:**

```python
import asyncio, time

async def fetch(name, delay):
    print(f"{name}: start")
    await asyncio.sleep(delay)     # yields control to event loop
    print(f"{name}: done after {delay}s")
    return name

async def main():
    start = time.perf_counter()
    results = await asyncio.gather(
        fetch("A", 1),
        fetch("B", 2),
        fetch("C", 1),
    )
    print(f"total: {time.perf_counter() - start:.1f}s")  # ~2s, not 4s

asyncio.run(main())
```

```
t=0.0s  event loop starts
        → schedules A, B, C into ready queue
        → runs A: prints "A: start", hits await sleep(1) → suspends, registers timer 1s
        → runs B: prints "B: start", hits await sleep(2) → suspends, registers timer 2s
        → runs C: prints "C: start", hits await sleep(1) → suspends, registers timer 1s
        → ready queue empty → polls I/O (nothing) → checks timers (nothing yet)
        → event loop sleeps (OS level) until earliest timer

t=1.0s  timers for A and C fire → both added to ready queue
        → runs A: prints "A: done after 1s", returns "A"
        → runs C: prints "C: done after 1s", returns "C"
        → ready queue empty → waits for B's timer

t=2.0s  timer for B fires
        → runs B: prints "B: done after 2s", returns "B"
        → gather() sees all done → main() continues
        → prints "total: 2.0s"
```

**Why `await` is the key:**
`await` is a yield point — it tells the event loop "I'm waiting for something, go do other work". Without `await`, the coroutine runs to completion without ever giving control back.

```python
# This blocks the ENTIRE event loop for 2 seconds — nothing else can run
async def bad():
    time.sleep(2)          # ← not async, no yield, freezes the loop

# This suspends only this coroutine — loop runs other tasks during the wait
async def good():
    await asyncio.sleep(2) # ← yield point, loop stays responsive
```

**Under the hood — `select`/`epoll`:**

When all coroutines are suspended waiting for I/O, the event loop calls `epoll_wait()` (Linux) or `kqueue` (macOS) — a single OS syscall that blocks until ANY of the watched file descriptors becomes ready. This is how one thread handles thousands of connections: it's not doing anything while waiting, just letting the OS tell it when work is available.

```python
# Conceptual implementation of the event loop core:
while True:
    # 1. Run everything that's ready
    for callback in ready_queue:
        callback()

    # 2. Ask OS: which sockets have data? (blocks until at least one does)
    timeout = next_timer_expiry()
    ready_fds = epoll.poll(timeout=timeout)

    # 3. Wake up coroutines whose I/O is ready
    for fd in ready_fds:
        waiting_coroutines[fd].send(None)   # resume

    # 4. Fire expired timers
    for timer in expired_timers():
        timer.callback()
```

**Key properties:**
- Single thread → no locks needed for coroutine-shared state (as long as no `await` between read and write)
- Cooperative → a coroutine that never `await`s starves everyone else
- I/O-bound → shines when tasks spend most time waiting for network/disk
- CPU-bound → use `run_in_executor` to offload to a thread/process pool

**Fan-out patterns:**

```python
# gather: all tasks, results in order, cancels siblings on exception
results = await asyncio.gather(fetch(url1), fetch(url2), fetch(url3))

# gather tolerant: collect errors instead of raising
results = await asyncio.gather(*tasks, return_exceptions=True)
errors = [r for r in results if isinstance(r, Exception)]

# wait: fine-grained — stop on first completion or first failure
done, pending = await asyncio.wait(
    [asyncio.create_task(t) for t in tasks],
    return_when=asyncio.FIRST_EXCEPTION
)
for p in pending:
    p.cancel()

# semaphore: limit concurrency (e.g. respect rate limits)
sem = asyncio.Semaphore(10)
async def fetch_limited(url):
    async with sem:
        return await fetch(url)
```

**Pitfalls:**
- Any blocking call (`requests.get`, `time.sleep`, CPU work) blocks the entire event loop
- Use `asyncio.to_thread()` to offload blocking calls:

```python
import asyncio, time

def blocking_work():
    time.sleep(2)          # would freeze the loop
    return "done"

async def main():
    result = await asyncio.to_thread(blocking_work)  # runs in a thread pool
    print(result)
```

---

## multiprocessing

Spawns separate Python interpreter processes — each has its own GIL, own memory space, own heap. True parallelism for CPU-bound work. Inter-process communication (IPC) via `Queue`, `Pipe`, or shared memory.

**When to use:** CPU-bound work — number crunching, image processing, ML inference, parsing large files.

```python
from multiprocessing import Pool
import os

def cpu_task(n):
    # simulate heavy computation
    return sum(i * i for i in range(n))

if __name__ == "__main__":   # required on Windows / macOS (spawn start method)
    with Pool(processes=os.cpu_count()) as pool:
        results = pool.map(cpu_task, [10_000_000] * 8)
    print(results)
```

**ProcessPoolExecutor (concurrent.futures API):**

```python
from concurrent.futures import ProcessPoolExecutor
import os

def crunch(data):
    return sum(x ** 2 for x in data)

chunks = [[range(1_000_000)] for _ in range(os.cpu_count())]

with ProcessPoolExecutor() as pool:
    results = list(pool.map(crunch, chunks))
```

**Sharing state between processes:**

```python
from multiprocessing import Value, Array, Lock

counter = Value('i', 0)   # shared integer
lock = Lock()

def increment(counter, lock):
    with lock:
        counter.value += 1
```

**IPC with Queue:**

```python
from multiprocessing import Process, Queue

def worker(q, n):
    q.put(n * n)

q = Queue()
procs = [Process(target=worker, args=(q, i)) for i in range(5)]
for p in procs: p.start()
for p in procs: p.join()

results = [q.get() for _ in range(5)]
print(results)
```

**Pitfalls:**
- Spawning processes is expensive (~100 ms); use pools, not per-task processes
- Objects must be picklable to cross the process boundary (lambdas, local classes — cannot)
- `if __name__ == "__main__"` guard is required on Windows/macOS

---

## Combining models

**asyncio + threads** — offload blocking calls without blocking the loop:

```python
async def main():
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(None, blocking_io_call)
    # None → default ThreadPoolExecutor
```

**asyncio + processes** — CPU work inside an async app:

```python
from concurrent.futures import ProcessPoolExecutor
import asyncio

async def main():
    loop = asyncio.get_event_loop()
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_heavy_function, data)
```

---

## Comparison summary

| | `threading` | `asyncio` | `multiprocessing` |
|---|---|---|---|
| Parallelism | No (GIL) | No (single thread) | Yes |
| Best for | Blocking I/O (legacy) | Async I/O (many conns) | CPU-bound |
| Overhead | ~8 MB / thread | ~1 KB / coroutine | ~50 MB / process |
| Shared state | Yes (needs locks) | Yes (no locks needed*) | No (IPC required) |
| Complexity | Medium | Medium | High |

*as long as there's no `await` between reads and writes to shared state

---
