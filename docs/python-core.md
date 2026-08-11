## 1. Python Core

### GIL (Global Interpreter Lock)

**Краткий ответ:**
Поток держит GIL и выполняет байт-код. Через интервал (`sys.getswitchinterval()`, по умолчанию 5 мс) интерпретатор проверяет, есть ли другие потоки, ожидающие GIL — если да, текущий поток обязан его отпустить. То есть переключение по таймеру, плюс поток может отпустить GIL раньше сам — например, при блокирующей операции вроде I/O.

!!! warning
    «Тик» — устаревшая терминология Python 2 (sys.getcheckinterval = 100 инструкций). В Python 3.2+ переключение по таймеру.

- `threading.Lock` — логическая защита кода от race conditions.
- GIL защищает внутренности интерпретатора (подсчёт ссылок), но НЕ защищает от логических ошибок в пользовательском коде.

**Подробнее:**
1. Только один поток выполняет байткод в каждый момент времени.
2. CPU-bound задачи — потоки не помогают. I/O-bound — помогают (GIL отпускается ядром ОС на время системного вызова I/O).
3. Обходы GIL:
- `multiprocessing` — отдельный интерпретатор на каждый процесс
- C-расширения (NumPy, Pillow) — явно освобождают GIL внутри тяжёлых вычислений
- free-threaded CPython (Python 3.13+, PEP 703, экспериментально)

!!! warning
    `asyncio` НЕ обходит GIL — он однопоточный и не нуждается в параллельных потоках.

🔗 [https://www.youtube.com/watch?v=AWX4JnAnjBE](https://www.youtube.com/watch?v=AWX4JnAnjBE)

---

### Управление памятью в Python

**Краткий ответ:**
arena (256 KB) → пул (4 KB) → блок (8–512 B для одного объекта).

!!! warning
    Минимальный блок — 8 байт (кратные 8: 8, 16, 24, ..., 512). Не 16.

Объект удаляется, когда счётчик ссылок = 0.
`sys.getrefcount()` возвращает n+1 (сама функция держит ссылку).
Числа -5..255 и интернированные строки — одиночки (один объект в памяти).
`__del__` вызывается перед удалением; `del` уменьшает refcount на 1.
Объекты > 512 B → память запрашивается напрямую у ОС.

**Как аллокатор выбирает арену:**
Новый объект — в самую ЗАПОЛНЕННУЮ арену. Цель: освобождать пустые арены как можно быстрее.

**Сборщик мусора:**
Помимо reference counting — cyclic GC (модуль `gc`) для обнаружения циклических ссылок.

---

### Генераторы, итераторы, итерируемые объекты

- Итерируемый объект — реализует `__iter__` (list, dict, str, set, range...)
- Итератор — реализует `__next__`; при исчерпании бросает `StopIteration`
- Генератор — функция с `yield`; автоматически является итератором; ленивые вычисления

**List comprehension vs Generator expression:**
- `[x**2 for x in nums]` — весь список сразу в памяти
- `(x**2 for x in nums)` — по одному элементу; для больших/бесконечных последовательностей

---

### Корутины и asyncio

**Краткий ответ:**
Корутина — функция, выполнение которой можно приостановить (`await`) и возобновить.
`asyncio` = кооперативная многозадачность: задачи сами решают, когда отдать управление.

**async vs thread vs process:**
- Корутины: самые лёгкие, один поток, нет переключения контекста ОС
- Потоки: поток ОС + GIL; хорошо для I/O-bound
- Процессы: отдельный интерпретатор, нет GIL; хорошо для CPU-bound

**Почему в asyncio меньше нужны Lock/Mutex:**
Один поток. Гонок данных нет, пока нет `await` между обращениями к shared state.

---

### Декораторы

Декоратор — callable, принимает функцию, возвращает другую. Синтаксический сахар для `func = decorator(func)`.

- `@functools.wraps(func)` — сохраняет `__name__`, `__doc__` и др.
- `@staticmethod` — без `self`, нет доступа к экземпляру или классу
- `@classmethod` — первый аргумент `cls` (класс); альтернативные конструкторы
- `@property` — превращает метод в управляемый атрибут (геттер/сеттер без явного вызова)

**Декоратор с параметрами (3 уровня вложенности):**

```python
def repeat(n):
    def decorator(func):
        def wrapper(*args, **kwargs):
            for _ in range(n):
                func(*args, **kwargs)
        return wrapper
    return decorator

@repeat(3)
def greet(name):
    print(f"Hello, {name}!")
```

---

### Контекстные менеджеры

Реализовать `__enter__` и `__exit__`.

- `__enter__`: вызывается при входе в `with`; возвращает управляемый объект
- `__exit__(exc_type, exc_value, traceback)`: вызывается при выходе; `True` — подавить исключение

Альтернатива: `@contextlib.contextmanager` с `yield`.

---

### Типы данных: list, dict, set, tuple

**dict — хеш-таблица:**
Коллизии — метод открытой адресации (random probing).
Python 3.6+: компактная раскладка — индексы отдельно, записи отдельно → экономия ~58% памяти.
Расширяется при 2/3 заполненности.

Оптимизация сравнения при коллизии:

```python
if id(key) == id(entity): return True    # один и тот же объект
if hash(key) != hash(entity): return False  # разные хеши
return key == entity                     # полное сравнение
```

**set:** неупорядоченный, неиндексируемый. Только hashable-элементы.

**is vs ==:**
- `is` — одна ли ячейка памяти (`id()`)
- `==` — равенство значений (`__eq__`)

**copy vs deepcopy:**
- `copy.copy()` — поверхностное: вложенные объекты — те же ссылки
- `copy.deepcopy()` — рекурсивно копирует все вложенные объекты

---

### __slots__

Заменяет `__dict__` экземпляра на фиксированный набор слотов → меньше памяти, быстрее доступ к атрибутам.
Минус: нет динамических атрибутов, сложнее множественное наследование.

```python
class Point:
    __slots__ = ('x', 'y')

    def __init__(self, x, y):
        self.x, self.y = x, y
```

---

### Дескрипторы

Объекты, реализующие `__get__`, `__set__`, `__delete__` — управляют доступом к атрибутам класса.
На них основаны `@property`, `@classmethod`, `@staticmethod`, поля ORM.

```python
class Validator:
    def __set_name__(self, owner, name):
        self.name = name

    def __get__(self, obj, objtype=None):
        return obj.__dict__.get(self.name)

    def __set__(self, obj, value):
        if not isinstance(value, int):
            raise TypeError(f"{self.name} must be int")
        obj.__dict__[self.name] = value

class MyModel:
    age = Validator()
```

---

### ABC (Abstract Base Classes)

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius

    def area(self) -> float:
        return 3.14159 * self.radius ** 2

# Shape()   → TypeError: нельзя создать экземпляр абстрактного класса
c = Circle(5)
print(c.area())  # 78.53975
```

### Protocol (структурная типизация)

`Protocol` решает ту же задачу без наследования — класс удовлетворяет протоколу, просто реализуя нужные методы (duck typing + статическая проверка типов).

```python
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> None: ...

class Circle:
    def draw(self) -> None:
        print("Drawing circle")

class Square:
    def draw(self) -> None:
        print("Drawing square")

def render(shape: Drawable) -> None:
    shape.draw()

render(Circle())  # ✓ — наследование не нужно
render(Square())  # ✓

# Ключевое отличие от ABC:
# ABC      — проверка при создании экземпляра (runtime)
# Protocol — проверка mypy/pyright на этапе type-check, без runtime-enforcement
```

---

### Аннотации типов (typing)

```python
int, str, list[int], dict[str, Any]
Optional[str]  # = str | None
TypeVar, Generic[T]  # обобщённые типы
Callable[[int, str], bool]
Protocol  # структурный интерфейс (duck typing + static check)
```

Проверка: `mypy`, `pyright`

---

### functools

```python
@lru_cache(maxsize=128)  # мемоизация с LRU-вытеснением; только для чистых функций
@cache                   # неограниченный кэш (Python 3.9+)

partial(func, *args)     # зафиксировать часть аргументов, вернуть новый callable
reduce(func, iterable)   # левая свёртка
```

---

### collections

```python
defaultdict(list)    # dict с фабрикой по умолчанию; нет KeyError
Counter(iterable)    # подсчёт элементов; поддерживает арифметику
deque(maxlen=N)      # O(1) append/pop с обоих концов; лучше list для очередей
OrderedDict          # сохраняет порядок вставки + move_to_end
namedtuple           # лёгкий иммутабельный record с именованными полями
```

---

### Алгоритмическая сложность

- list: O(1) доступ по индексу, O(n) поиск, O(1) амортизированный append
- dict/set: O(1) средний поиск/вставка, O(n) худший случай
- Big O: O(1) → O(log n) → O(n) → O(n log n) → O(n²) → O(2ⁿ)

---

### type() и Метаклассы

**Краткий ответ:**
`type` — метакласс для создания классов. Все классы в Python — объекты, созданные через `type`.
`type(name, bases, attrs)` — создаёт новый класс динамически.

```python
class MyMeta(type):
    def __new__(cls, name, bases, attrs):
        attrs['custom_attr'] = 42
        return super().__new__(cls, name, bases, attrs)

class MyClass(metaclass=MyMeta):
    pass

print(MyClass.custom_attr)  # 42
```

Применение: ORM (Django Models), API-фреймворки, валидация классов.

---

### Динамический импорт модуля

```python
import importlib.util

def import_module_from_path(path):
    spec = importlib.util.spec_from_file_location("module", path)
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)
    return module
```

---
