# Threading vs asyncio vs Multiprocessing

## Суть проблемы

В Python три модели конкурентности. Неправильный выбор стоит производительности; правильный — разница в 10 раз.

Корень различий — **GIL**: только один поток выполняет байт-код Python в каждый момент. Это делает `threading` бесполезным для CPU-bound задач, но нормальным для I/O-bound (GIL отпускается во время системных вызовов I/O).

---

## Быстрый выбор

| Задача | Используй |
|---|---|
| Много сетевых запросов / запросов в БД | `asyncio` |
| Блокирующий I/O, который нельзя сделать async | `threading` |
| Тяжёлые вычисления (ML, обработка изображений) | `multiprocessing` |
| CPU + I/O вместе | `multiprocessing` + `asyncio` внутри каждого воркера |

---

## threading

Поток — единица выполнения на уровне ОС, разделяющая общую память. GIL означает, что только один поток выполняет байт-код Python одновременно, но потоки действительно выполняются параллельно во время ожидания I/O.

**Когда использовать:** блокирующие I/O-вызовы, которые нельзя `await` (legacy-библиотеки, `subprocess`).

```python
import threading
import requests

results = {}

def fetch(url):
    results[url] = requests.get(url).status_code  # блокирующий — GIL отпускается при ожидании сети

urls = ["https://httpbin.org/get"] * 5
threads = [threading.Thread(target=fetch, args=(url,)) for url in urls]
for t in threads: t.start()
for t in threads: t.join()
print(results)
```

**ThreadPoolExecutor (предпочтительнее):**

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

**Разделяемое состояние требует блокировок:**

```python
import threading

counter = 0
lock = threading.Lock()

def increment():
    global counter
    with lock:
        counter += 1  # read-modify-write не атомарен

threads = [threading.Thread(target=increment) for _ in range(1000)]
for t in threads: t.start()
for t in threads: t.join()
print(counter)  # 1000 ✓ (без lock: непредсказуемо)
```

**Подводные камни:**
- Race conditions при изменении общего состояния
- Deadlock при захвате нескольких блокировок в разном порядке
- Overhead памяти: ~8 МБ стека на поток по умолчанию в Linux

---

## asyncio

Однопоточная кооперативная многозадачность. Корутина добровольно отдаёт управление на каждом `await`, позволяя event loop запустить другую корутину. Нет конкуренции за GIL, нет переключений контекста ОС — очень дёшево.

**Когда использовать:** высококонкурентный I/O (тысячи HTTP-запросов, WebSocket-соединений, запросов в БД с async-драйвером).

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

**Event loop: что происходит на самом деле**

```
event loop
  ├── корутина A: await network_call()  → приостановлена, loop идёт дальше
  ├── корутина B: await db_query()      → приостановлена, loop идёт дальше
  ├── корутина C: await asyncio.sleep() → приостановлена, loop идёт дальше
  └── network_call() завершился → A возобновляется
```

Параллелизма нет — но и ожидания нет. 1000 одновременных запросов стоят примерно столько же, сколько 10.

**Паттерны fan-out:**

```python
# gather: все задачи, результаты по порядку, отменяет «соседей» при исключении
results = await asyncio.gather(fetch(url1), fetch(url2), fetch(url3))

# gather толерантный: собирать ошибки вместо исключения
results = await asyncio.gather(*tasks, return_exceptions=True)
errors = [r for r in results if isinstance(r, Exception)]

# wait: тонкий контроль — остановить при первом завершении или первой ошибке
done, pending = await asyncio.wait(
    [asyncio.create_task(t) for t in tasks],
    return_when=asyncio.FIRST_EXCEPTION
)
for p in pending:
    p.cancel()

# semaphore: ограничить конкурентность (например, соблюдать rate limit)
sem = asyncio.Semaphore(10)
async def fetch_limited(url):
    async with sem:
        return await fetch(url)
```

**Подводные камни:**
- Любой блокирующий вызов (`requests.get`, `time.sleep`, CPU-работа) замораживает весь event loop
- Используй `asyncio.to_thread()` для переноса блокирующих вызовов в поток:

```python
import asyncio, time

def blocking_work():
    time.sleep(2)          # заморозило бы loop
    return "done"

async def main():
    result = await asyncio.to_thread(blocking_work)  # выполняется в пуле потоков
    print(result)
```

---

## multiprocessing

Запускает отдельные процессы интерпретатора Python — у каждого свой GIL, своя память, своя куча. Настоящий параллелизм для CPU-bound задач. Межпроцессное взаимодействие (IPC) через `Queue`, `Pipe` или разделяемую память.

**Когда использовать:** CPU-bound работа — числовые вычисления, обработка изображений, ML-инференс, парсинг больших файлов.

```python
from multiprocessing import Pool
import os

def cpu_task(n):
    return sum(i * i for i in range(n))

if __name__ == "__main__":   # обязательно на Windows / macOS (spawn start method)
    with Pool(processes=os.cpu_count()) as pool:
        results = pool.map(cpu_task, [10_000_000] * 8)
    print(results)
```

**ProcessPoolExecutor (API concurrent.futures):**

```python
from concurrent.futures import ProcessPoolExecutor
import os

def crunch(data):
    return sum(x ** 2 for x in data)

chunks = [range(1_000_000) for _ in range(os.cpu_count())]

with ProcessPoolExecutor() as pool:
    results = list(pool.map(crunch, chunks))
```

**IPC через Queue:**

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

**Подводные камни:**
- Запуск процесса дорог (~100 мс); используй пулы, не процессы на каждую задачу
- Объекты должны быть picklable для передачи между процессами (лямбды, локальные классы — нельзя)
- Обязательна защита `if __name__ == "__main__"` на Windows/macOS

---

## Комбинирование моделей

**asyncio + потоки** — выгрузить блокирующий вызов без заморозки loop:

```python
async def main():
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(None, blocking_io_call)
    # None → стандартный ThreadPoolExecutor
```

**asyncio + процессы** — CPU-работа внутри async-приложения:

```python
from concurrent.futures import ProcessPoolExecutor
import asyncio

async def main():
    loop = asyncio.get_event_loop()
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_heavy_function, data)
```

---

## Итоговое сравнение

| | `threading` | `asyncio` | `multiprocessing` |
|---|---|---|---|
| Параллелизм | Нет (GIL) | Нет (один поток) | Да |
| Лучше для | Блокирующий I/O (legacy) | Async I/O (много соединений) | CPU-bound |
| Overhead | ~8 МБ / поток | ~1 КБ / корутина | ~50 МБ / процесс |
| Разделяемое состояние | Да (нужны lock-и) | Да (lock не нужен*) | Нет (нужен IPC) |
| Сложность | Средняя | Средняя | Высокая |

*при условии что между чтением и записью разделяемого состояния нет `await`

---
