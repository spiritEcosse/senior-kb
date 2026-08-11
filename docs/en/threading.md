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

```
event loop
  ├── coroutine A: await network_call()  → suspended, loop moves on
  ├── coroutine B: await db_query()      → suspended, loop moves on
  ├── coroutine C: await asyncio.sleep() → suspended, loop moves on
  └── network_call() completes → A resumes
```

No parallelism — but no waiting either. 1000 concurrent requests cost roughly the same as 10.

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
