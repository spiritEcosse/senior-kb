## 3. Параллелизм и асинхронность

### Thread vs Process vs Async


Thread            Process         Coroutine (async)
Изоляция памяти     Нет (shared)   Да                 Нет (один поток)
GIL                            Да                   Нет                Да (уступает сам)
Подходит для          I/O-bound        CPU-bound   I/O-bound (много соединений)
Стоимость               Средняя          Высокая       Очень низкая

### Thread Pool и Process Pool

ThreadPoolExecutor (concurrent.futures) — для I/O-bound.
ProcessPoolExecutor / multiprocessing.Pool — для CPU-bound; свой GIL в каждом процессе.

### asyncio: ключевые паттерны

asyncio.gather(*coros) — запустить корутины параллельно, вернуть результаты по порядку.
asyncio.create_task(coro) — запланировать корутину как фоновую задачу.
asyncio.wait_for(coro, timeout) — добавить таймаут.
asyncio.Lock(), asyncio.Semaphore() — примитивы синхронизации для async-кода.

---

### Профилировщики

- `Intel VTune`, `Valgrind` — для C-расширений и многопоточного кода
- `memory_profiler` — профилирование потребления памяти в Python
- `cProfile`, `py-spy`, `line_profiler` — для чистого Python-кода

---
