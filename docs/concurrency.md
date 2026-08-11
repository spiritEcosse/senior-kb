## 3. Параллелизм и асинхронность

### Thread vs Process vs Async

| | Thread | Process | Coroutine (async) |
|---|---|---|---|
| Изоляция памяти | Нет (shared) | Да | Нет (один поток) |
| GIL | Да | Нет | Да (уступает сам) |
| Подходит для | I/O-bound | CPU-bound | I/O-bound (много соединений) |
| Стоимость | Средняя | Высокая | Очень низкая |

### Thread Pool и Process Pool

`ThreadPoolExecutor` (`concurrent.futures`) — для I/O-bound.
`ProcessPoolExecutor` / `multiprocessing.Pool` — для CPU-bound; свой GIL в каждом процессе.

### asyncio: ключевые паттерны

```python
# gather — запустить параллельно, вернуть результаты по порядку
results = await asyncio.gather(fetch(url1), fetch(url2), fetch(url3))

# gather с return_exceptions — не падать при ошибке одного
results = await asyncio.gather(*tasks, return_exceptions=True)
for r in results:
    if isinstance(r, Exception):
        logger.error(r)

# create_task — фоновая задача (не блокирует)
task = asyncio.create_task(background_job())

# wait — точный контроль: FIRST_COMPLETED, FIRST_EXCEPTION, ALL_COMPLETED
done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_EXCEPTION)

# wait_for — таймаут
data = await asyncio.wait_for(fetch(url), timeout=5.0)

# Lock, Semaphore — синхронизация
async with asyncio.Semaphore(10):   # макс. 10 одновременных
    await fetch(url)
```

**gather vs wait:**

| | `gather` | `wait` |
|---|---|---|
| Вход | корутины | Task-объекты |
| Отмена при ошибке | да (по умолчанию) | нет |
| Когда использовать | простой fan-out | точный контроль |

### asyncio.gather vs asyncio.create_task

```python
# create_task — запланировать немедленно, не ждать
task = asyncio.create_task(fetch(url))   # стартует сразу
# ... другой код ...
result = await task                       # дождаться результата

# await coroutine — выполнить последовательно (не параллельно!)
result = await fetch(url)                 # блокирует до завершения
```

### Проблема позднего связывания в замыканиях

```python
# Проблема: все функции захватывают одну переменную i
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])  # [2, 2, 2] — не [0, 1, 2]!

# Исправление: зафиксировать значение через аргумент по умолчанию
funcs = [lambda i=i: i for i in range(3)]
print([f() for f in funcs])  # [0, 1, 2] ✓
```

### Профилировщики

- `cProfile` — встроенный, детальный анализ вызовов
- `py-spy` — sampling profiler, работает без изменения кода, подходит для production
- `line_profiler` — профилирование построчно
- `memory_profiler` — потребление памяти в Python
- `Intel VTune`, `Valgrind` — для C-расширений и многопоточного кода

```python
# py-spy: запуск без изменения кода
# py-spy record -o profile.svg --pid 12345
```

---
