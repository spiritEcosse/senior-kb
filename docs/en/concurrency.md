## 3. Concurrency and Async

### Thread vs Process vs Async

|                  | Thread          | Process        | Coroutine (async)         |
|------------------|-----------------|----------------|---------------------------|
| Memory isolation | No (shared)     | Yes            | No (single thread)        |
| GIL              | Yes             | No             | Yes (yields voluntarily)  |
| Best for         | I/O-bound       | CPU-bound      | I/O-bound (many conns)    |
| Cost             | Medium          | High           | Very low                  |

### Thread Pool and Process Pool

`ThreadPoolExecutor` (`concurrent.futures`) — for I/O-bound.
`ProcessPoolExecutor` / `multiprocessing.Pool` — for CPU-bound; own GIL per process.

### asyncio: Key Patterns

`asyncio.gather(*coros)` — run coroutines concurrently, return results in order.
`asyncio.create_task(coro)` — schedule a coroutine as a background task.
`asyncio.wait_for(coro, timeout)` — add a timeout.
`asyncio.Lock()`, `asyncio.Semaphore()` — synchronisation primitives for async code.

---
