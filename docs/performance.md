## 13. Производительность Python vs C++

### Почему Python медленнее

| Фактор | Python | C++ |
|---|---|---|
| Выполнение | Байткод интерпретируется CPython | Компилируется в машинный код |
| Типы | Проверяются в runtime (динамические) | Проверяются на этапе компиляции |
| Память | Каждое значение — объект кучи с refcount | Стек/куча, value types, нет GC overhead |
| Потоки | GIL ограничивает параллелизм | Настоящий многоядерный параллелизм |
| Диспетчеризация | Virtual dispatch при каждом обращении к атрибуту | Инлайнинг, девиртуализация компилятором |

---

### Сначала профилируй — никогда не угадывай

```python
# cProfile — встроенный, статистика на уровне вызовов
python -m cProfile -s cumulative my_script.py

import cProfile
cProfile.run('my_function()', sort='cumulative')

# line_profiler — построчно (pip install line-profiler)
@profile
def slow_function():
    ...
kernprof -l -v my_script.py

# memory_profiler — потребление памяти построчно
@profile
def memory_heavy():
    ...
python -m memory_profiler my_script.py

# py-spy — sampling profiler, без изменений кода, безопасен в production
py-spy record -o profile.svg --pid 12345
py-spy top --pid 12345          # живой top-like вид
```

---

### Типичные оптимизации

**1. Векторизация через NumPy — избегай Python-циклов по числовым данным:**

```python
import numpy as np

# Медленно: Python-цикл
result = [x ** 2 for x in data]       # ~1.2s для 10M элементов

# Быстро: NumPy векторизация
result = np.array(data) ** 2          # ~0.01s
```

**2. Встроенные функции и stdlib — реализованы на C:**

```python
# Медленно
total = 0
for x in data:
    total += x

# Быстро
total = sum(data)          # C-цикл внутри
```

**3. Избегай повторных поисков атрибутов в горячих циклах:**

```python
# Медленно — ищет math.sqrt при каждой итерации
import math
for x in data:
    math.sqrt(x)

# Быстро — локальная ссылка
from math import sqrt
for x in data:
    sqrt(x)
```

**4. Генераторы для больших последовательностей:**

```python
# Строит весь список в памяти
total = sum([x ** 2 for x in range(10_000_000)])

# Генератор — по одному элементу
total = sum(x ** 2 for x in range(10_000_000))
```

**5. Конкатенация строк:**

```python
# Медленно: новый строковый объект на каждой итерации
result = ""
for s in parts:
    result += s

# Быстро: одно выделение памяти
result = "".join(parts)
```

---

### Инструменты ускорения

| Инструмент | Подход | Ускорение |
|---|---|---|
| NumPy | C-массивы + BLAS | 10–100× для числовых |
| Cython | Компиляция Python в C | 10–100× |
| Numba | JIT-компиляция через LLVM | 10–200× для числовых |
| PyPy | Tracing JIT интерпретатор | 3–10× для общего Python |
| multiprocessing | Обход GIL через процессы | N× (N = кол-во ядер) |
| asyncio | Конкурентный I/O, один поток | Высокая конкурентность |
| C-расширение (cffi, ctypes) | Вызов C/Rust библиотек | Близко к native |
| pyo3 | Python-расширения на Rust | Близко к native |

---
