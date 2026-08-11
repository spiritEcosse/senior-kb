## 3. Concurrency and Async

### Thread vs Process vs Async

| | Thread | Process | Coroutine (async) |
|---|---|---|---|
| Memory isolation | No (shared) | Yes | No (single thread) |
| GIL | Yes | No | Yes (yields voluntarily) |
| Best for | I/O-bound | CPU-bound | I/O-bound (many conns) |
| Cost | Medium | High | Very low |

### Thread Pool and Process Pool

`ThreadPoolExecutor` (`concurrent.futures`) — for I/O-bound.
`ProcessPoolExecutor` / `multiprocessing.Pool` — for CPU-bound; own GIL per process.

### asyncio: Key Patterns

```python
# gather — run concurrently, return results in order
results = await asyncio.gather(fetch(url1), fetch(url2), fetch(url3))

# gather with return_exceptions — don't fail on one error
results = await asyncio.gather(*tasks, return_exceptions=True)
for r in results:
    if isinstance(r, Exception):
        logger.error(r)

# create_task — background task (non-blocking)
task = asyncio.create_task(background_job())

# wait — fine-grained control: FIRST_COMPLETED, FIRST_EXCEPTION, ALL_COMPLETED
done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_EXCEPTION)

# wait_for — timeout
data = await asyncio.wait_for(fetch(url), timeout=5.0)

# Lock, Semaphore — synchronisation
async with asyncio.Semaphore(10):   # max 10 concurrent
    await fetch(url)
```

**gather vs wait:**

| | `gather` | `wait` |
|---|---|---|
| Input | coroutines | Task objects |
| Cancels on error | yes (default) | no |
| Use when | simple fan-out | fine-grained control |

### asyncio.gather vs asyncio.create_task

```python
# create_task — schedules immediately, doesn't block
task = asyncio.create_task(fetch(url))   # starts immediately
# ... other code ...
result = await task                       # wait for result

# await coroutine — executes sequentially (NOT parallel!)
result = await fetch(url)                 # blocks until done
```

### Late Binding in Closures

```python
# Problem: all functions capture the same variable i
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])  # [2, 2, 2] — not [0, 1, 2]!

# Fix: bind the value via a default argument
funcs = [lambda i=i: i for i in range(3)]
print([f() for f in funcs])  # [0, 1, 2] ✓
```

### Profilers

- `cProfile` — built-in, detailed call analysis
- `py-spy` — sampling profiler, works without code changes, production-safe
- `line_profiler` — line-by-line profiling
- `memory_profiler` — Python memory usage profiling
- `Intel VTune`, `Valgrind` — for C extensions and multithreaded code

```python
# py-spy: attach to running process without code changes
# py-spy record -o profile.svg --pid 12345
```

---
