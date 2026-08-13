## 1. Python Core

### GIL (Global Interpreter Lock)

**Short answer:**
A thread holds the GIL and executes bytecode. After the interval (`sys.getswitchinterval()`, default 5 ms), the interpreter checks whether other threads are waiting for the GIL — if so, the current thread is forced to release it. So switching is timer-based, but a thread can also release the GIL earlier on its own, for example during a blocking I/O operation.

!!! warning
    "Tick" is outdated Python 2 terminology (`sys.getcheckinterval` = 100 instructions). In Python 3.2+ switching is timer-based.

- `threading.Lock` — logical protection of code from race conditions.
- The GIL protects interpreter internals (reference counting) but does NOT protect against logical errors in user code.

**In depth:**
1. Only one thread executes bytecode at any given moment.
2. CPU-bound tasks — threads don't help. I/O-bound — they do (GIL is released by the OS kernel during I/O syscalls).
3. GIL workarounds:
- `multiprocessing` — separate interpreter per process
- C extensions (NumPy, Pillow) — explicitly release the GIL during heavy computation
- Free-threaded CPython (Python 3.13+, PEP 703, experimental)

!!! warning
    `asyncio` does NOT bypass the GIL — it is single-threaded and doesn't need parallel threads.

🔗 [https://www.youtube.com/watch?v=AWX4JnAnjBE](https://www.youtube.com/watch?v=AWX4JnAnjBE)

---

### Memory Management in Python

**Short answer:**
arena (256 KB) → pool (4 KB) → block (8–512 B per object).

!!! warning
    Minimum block size is 8 bytes (multiples of 8: 8, 16, 24, ..., 512). Not 16.

An object is destroyed when its reference count reaches 0.
`sys.getrefcount()` returns n+1 (the function itself holds a reference).
Integers -5..255 and interned strings are singletons (one object in memory).
`__del__` is called before deletion; `del` decrements refcount by 1.
Objects > 512 B → memory requested directly from the OS.

**How the allocator picks an arena:**
New object goes into the most OCCUPIED arena. Goal: free empty arenas as fast as possible.

**Garbage collector:**
Beyond reference counting — cyclic GC (`gc` module) to detect circular references.

---

### Generators, Iterators, Iterables

- Iterable — implements `__iter__` (list, dict, str, set, range...)
- Iterator — implements `__next__`; raises `StopIteration` when exhausted
- Generator — function with `yield`; automatically an iterator; lazy evaluation

**`yield` vs `return` vs `yield from`:**
- `return value` — exits the function immediately, returns a value (not a generator)
- `yield value` — suspends the function, emits a value, resumes on next `next()` call
- `yield from iterable` — delegates to another iterable/generator, exhausting it fully; equivalent to `for item in iterable: yield item` but more efficient

```python
# yield from — correct way to recurse in a generator
def flatten(data):
    for item in data:
        if isinstance(item, list):
            yield from flatten(item)  # delegates to recursive generator
        else:
            yield item

list(flatten([1, [2, [3, 4]], 5]))  # [1, 2, 3, 4, 5]

# yield — emits the generator object itself (wrong for recursion)
# return — exits immediately, only first item processed
```

**List comprehension vs Generator expression:**
- `[x**2 for x in nums]` — full list in memory at once
- `(x**2 for x in nums)` — one element at a time; better for large/infinite sequences

---

### Coroutines and asyncio

**Short answer:**
A coroutine is a function whose execution can be suspended (`await`) and resumed.
`asyncio` = cooperative multitasking: tasks decide themselves when to yield control.

**async vs thread vs process:**
- Coroutines: lightest, single thread, no OS context switching
- Threads: OS thread + GIL; good for I/O-bound
- Processes: separate interpreter, no GIL; good for CPU-bound

**Why asyncio needs fewer Lock/Mutex:**
Single thread. No data races as long as there's no `await` between accesses to shared state.

---

### Decorators

A decorator is a callable that takes a function and returns another. Syntactic sugar for `func = decorator(func)`.

`@functools.wraps(func)` — preserves `__name__`, `__doc__`, etc.
`@staticmethod` — no `self`, no access to instance or class
`@classmethod` — first argument is `cls` (class); used for alternative constructors
`@property` — turns a method into a managed attribute (getter/setter without explicit call)

**Decorator with parameters (3 nesting levels):**

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

### Context Managers

Implement `__enter__` and `__exit__`.
- `__enter__`: called on `with` entry; returns the managed object
- `__exit__(exc_type, exc_value, traceback)`: called on exit; return `True` to suppress the exception

Alternative: `@contextlib.contextmanager` with `yield` — cleaner for simple cases, no class needed:

```python
from contextlib import contextmanager
import time

@contextmanager
def timer(label: str):
    start = time.perf_counter()
    try:
        yield                          # everything inside 'with' runs here
    finally:
        elapsed = time.perf_counter() - start
        print(f"{label}: {elapsed:.3f}s")

with timer("db query"):
    results = db.execute("SELECT ...")

# DB connection example — yield the resource, cleanup in finally
@contextmanager
def get_connection(url: str):
    conn = connect(url)
    try:
        yield conn                     # 'as conn' in the with statement
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()                   # always runs, even on exception

with get_connection(DATABASE_URL) as conn:
    conn.execute("INSERT ...")
    conn.commit()

# How it maps to __enter__ / __exit__:
# everything before yield  → __enter__
# yield value              → what 'as' receives
# finally block            → __exit__ (always)
# except block             → __exit__ when exc_type is not None
```

---

### Data Types: list, dict, set, tuple

**dict — hash table:**
Collisions handled via open addressing (random probing).
Python 3.6+: compact layout — indices separate from entries → ~58% memory savings.
Resizes at 2/3 capacity.

Comparison optimisation on collision:
```python
if id(key) == id(entity): return True
if hash(key) != hash(entity): return False
return key == entity
```

**set:** unordered, unindexed. Only hashable elements.

**is vs ==:**
- `is` — same memory address (`id()`)
- `==` — value equality (`__eq__`)

**copy vs deepcopy:**
- `copy.copy()` — shallow: nested objects are shared references
- `copy.deepcopy()` — recursively copies all nested objects

---

### __slots__

Replaces the instance `__dict__` with a fixed set of slots → less memory, faster attribute access.
Downside: no dynamic attributes, more complex multiple inheritance.

```python
class Point:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x, self.y = x, y
```

---

### Descriptors

Objects implementing `__get__`, `__set__`, `__delete__` — control access to class attributes.
`@property`, `@classmethod`, `@staticmethod`, ORM fields are all built on descriptors.

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

# Shape()   → TypeError: can't instantiate abstract class
c = Circle(5)
print(c.area())  # 78.53975
```

### Protocol (structural typing)

`Protocol` achieves the same goal without inheritance — a class satisfies a Protocol simply by implementing the required methods (duck typing + static type checking).

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

render(Circle())  # ✓ — no inheritance required
render(Square())  # ✓

# Key difference from ABC:
# ABC  — enforced at instantiation time (runtime)
# Protocol — checked by mypy/pyright at type-check time, no runtime enforcement
```

---

### Type Annotations (typing)

```python
int, str, list[int], dict[str, Any]
Optional[str]           # = str | None
Callable[[int, str], bool]
Protocol                # structural interface (duck typing + static check)
```

#### TypeVar and Generic

`TypeVar` declares a placeholder type that gets resolved at call time. `Generic[T]` makes a class parametric over that placeholder.

```python
from typing import TypeVar, Generic

T = TypeVar('T')

# Generic function — T is inferred from the argument
def first(items: list[T]) -> T:
    return items[0]

first([1, 2, 3])      # T = int   → returns int
first(['a', 'b'])     # T = str   → returns str

# Generic class — T is set when the class is instantiated
class Box(Generic[T]):
    def __init__(self, value: T) -> None:
        self.value = value

    def unwrap(self) -> T:
        return self.value

box: Box[int] = Box(42)
print(box.unwrap())   # 42
```

**Bounded TypeVar** — restrict T to a specific type or its subclasses:

```python
from typing import TypeVar
from numbers import Number

N = TypeVar('N', bound=Number)   # T must be Number or a subclass

def add(a: N, b: N) -> N:
    return a + b   # type: ignore

add(1, 2)       # ✓ int is a Number
add(1.0, 2.0)   # ✓ float is a Number
# add("a", "b") # ✗ mypy error: str is not a Number
```

**Python 3.12+ new syntax** — identical semantics, no import needed:

```python
# Old way
T = TypeVar('T')
class Box(Generic[T]): ...

# New way (Python 3.12+) — same semantics, cleaner syntax
class Box[T]: ...
class Pair[K, V]: ...
class Repository[S: Serializable]: ...  # bound
```

Checkers: `mypy`, `pyright`

---

### functools

#### @lru_cache and @cache

```python
from functools import lru_cache, cache

# lru_cache — bounded: evicts least-recently-used when maxsize is reached
@lru_cache(maxsize=128)
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

print(fibonacci(50))          # fast — results are cached
print(fibonacci.cache_info()) # CacheInfo(hits=48, misses=51, maxsize=128, currsize=51)
fibonacci.cache_clear()       # reset the cache

# cache — unbounded (Python 3.9+), equivalent to lru_cache(maxsize=None)
@cache
def factorial(n: int) -> int:
    return 1 if n == 0 else n * factorial(n - 1)
```

!!! warning
    Only use on **pure functions** — same arguments must always produce the same result. Never cache functions with side effects or that depend on mutable state.

#### partial

Fixes some arguments of a function, returning a new callable with fewer parameters:

```python
from functools import partial

def power(base, exponent):
    return base ** exponent

square = partial(power, exponent=2)
cube   = partial(power, exponent=3)

print(square(5))   # 25
print(cube(3))     # 27

# Useful for callbacks and higher-order functions
numbers = [1, 2, 3, 4, 5]
multiply_by_10 = partial(map, partial(power, exponent=1))  # trivial example
print(list(map(square, numbers)))  # [1, 4, 9, 16, 25]
```

#### reduce

Left fold — applies a binary function cumulatively to reduce a sequence to a single value:

```python
from functools import reduce

# Sum without built-in sum()
result = reduce(lambda acc, x: acc + x, [1, 2, 3, 4, 5])
print(result)   # 15  →  ((((1+2)+3)+4)+5)

# Product
product = reduce(lambda acc, x: acc * x, [1, 2, 3, 4, 5])
print(product)  # 120

# Flatten nested list
nested = [[1, 2], [3, 4], [5, 6]]
flat = reduce(lambda acc, x: acc + x, nested)
print(flat)     # [1, 2, 3, 4, 5, 6]

# With initial value (avoids error on empty sequence)
reduce(lambda acc, x: acc + x, [], 0)   # 0 (not an error)
```

**When to prefer alternatives:**
- `sum()` over `reduce(lambda a, b: a + b, ...)`
- `itertools.chain.from_iterable()` over `reduce` for flattening
- Use `reduce` when there's no built-in for the specific accumulation logic

---

### collections

`defaultdict(list)` — dict with a default factory; no `KeyError`
`Counter(iterable)` — element counting; supports arithmetic
`deque(maxlen=N)` — O(1) append/pop from both ends; better than list for queues
`OrderedDict` — preserves insertion order + `move_to_end`
`namedtuple` — lightweight immutable record with named fields

---

### Algorithmic Complexity

- list: O(1) index access, O(n) search, O(1) amortised append
- dict/set: O(1) average search/insert, O(n) worst case
- Big O: O(1) → O(log n) → O(n) → O(n log n) → O(n²) → O(2ⁿ)

---

### type() and Metaclasses

**Short answer:**
`type` is the metaclass for creating classes. All classes in Python are objects created via `type`.
`type(name, bases, attrs)` — creates a new class dynamically.

**Metaclasses:**
A metaclass controls class creation. It can define `__new__` or `__init__` (or both).

**`__new__` vs `__init__` in a metaclass:**
- `__new__(cls, name, bases, attrs)` — creates and returns the class object; `cls` is the metaclass
- `__init__(cls, name, bases, attrs)` — called after the class is created; `cls` is already the new class

```python
class MyMeta(type):
    def __new__(cls, name, bases, attrs):
        attrs['custom_attr'] = 42
        return super().__new__(cls, name, bases, attrs)

class MyClass(metaclass=MyMeta):
    pass

print(MyClass.custom_attr)  # 42
```

**Using `__init__` instead — class already exists as `cls`:**
```python
class Registry(type):
    all = {}

    def __init__(cls, name, bases, attrs):
        super().__init__(name, bases, attrs)
        Registry.all[name] = cls  # cls is the newly created class
```

**Key distinction:** In a regular class `__init__` receives `self` (an instance). In a metaclass `__init__` receives `cls` (a class) — because classes are instances of their metaclass.

Use cases: ORM (Django Models), API frameworks, class validation, auto-registration.

---

### Dynamic Module Import

```python
import importlib.util

def import_module_from_path(path):
    spec = importlib.util.spec_from_file_location("module", path)
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)
    return module
```

---

### Exception chaining and custom hierarchies

```python
# raise X from Y — explicit chaining (preserves original traceback)
try:
    result = db.query(sql)
except DatabaseError as e:
    raise ServiceUnavailableError("DB unreachable") from e
# → ServiceUnavailableError: DB unreachable
#   The above exception was the direct cause of the following exception:
#   DatabaseError: connection refused

# raise X from None — suppress original exception context
try:
    value = int(user_input)
except ValueError:
    raise ValidationError("Expected an integer") from None

# Custom exception hierarchy
class AppError(Exception):
    """Base for all application errors."""

class ValidationError(AppError):
    def __init__(self, field: str, message: str):
        self.field = field
        super().__init__(f"{field}: {message}")

class NotFoundError(AppError):
    pass

class ServiceUnavailableError(AppError):
    pass

# Catch at the right level
try:
    process(data)
except ValidationError as e:
    return 422, {"field": e.field, "error": str(e)}
except NotFoundError:
    return 404, {"error": "not found"}
except AppError:
    return 500, {"error": "internal error"}
```

---

### contextlib: suppress and ExitStack

```python
from contextlib import suppress, ExitStack

# suppress — silently ignore specific exceptions
with suppress(FileNotFoundError):
    os.remove('tmp.txt')   # no error if file doesn't exist

# ExitStack — dynamic number of context managers
def process_files(paths: list[str]):
    with ExitStack() as stack:
        files = [stack.enter_context(open(p)) for p in paths]
        # all files closed automatically, even if one fails mid-loop
        for f in files:
            process(f.read())

# Useful when you don't know at write-time how many CMs you need
```

---

### weakref

A weak reference doesn't prevent an object from being garbage collected.

```python
import weakref

class Cache:
    def __init__(self):
        self._data = {}   # strong refs — objects kept alive by cache

class WeakCache:
    def __init__(self):
        self._data = weakref.WeakValueDictionary()   # weak refs — GC can collect values

obj = MyObject()
cache = WeakCache()
cache._data['key'] = obj
del obj                    # refcount → 0; GC collects it
print(cache._data.get('key'))  # None — automatically removed

# Use cases:
# - Caches that shouldn't prevent GC (avoid memory leaks)
# - Observer/callback registries (don't keep dead objects alive)
# - __weakref__ must be in __slots__ if you use __slots__
```

---

### Python internals: bytecode and import system

```python
# dis — inspect bytecode
import dis

def add(a, b):
    return a + b

dis.dis(add)
#   2           0 RESUME          0
#   3           2 LOAD_FAST       0 (a)
#               4 LOAD_FAST       1 (b)
#               6 BINARY_OP      0 (+)
#              10 RETURN_VALUE

# Import system
import sys
print(sys.modules.keys())          # all currently imported modules
print(sys.path)                    # directories Python searches for modules

# __all__ — controls what 'from module import *' exports
# mymodule.py
__all__ = ['PublicClass', 'public_function']   # private_helper not exported

# Namespace packages (PEP 420) — directory without __init__.py
# Allows splitting a package across multiple directories/distributions
```

---

---

### Type hints — do they impact runtime speed?

**Short answer: No at runtime, but yes for `__pycache__` generation.**

Type hints are **purely static** — they exist only for tools like `mypy`, `pyright`, and IDEs. At runtime, Python stores them in `__annotations__` but does **not enforce them**.

```python
def add(x: int, y: int) -> int:
    return x + y

add("hello", "world")   # works fine at runtime — no error
print(add.__annotations__)  # {'x': <class 'int'>, 'y': <class 'int'>, 'return': <class 'int'>}
```

**Runtime cost is negligible:**
- Annotations are parsed once at module import and stored as strings (with `from __future__ import annotations`) or as evaluated objects
- No checking happens on each function call
- `from __future__ import annotations` (PEP 563) defers evaluation → annotations stored as strings → zero evaluation cost

```python
from __future__ import annotations   # all annotations become strings, never evaluated

def process(items: list[User]) -> dict[str, int]:  # 'list[User]' stored as string
    ...
```

**Do type hints affect `__pycache__` / bytecode?**

Yes — indirectly. Python compiles every `.py` file to bytecode (`.pyc` in `__pycache__`) regardless of whether it has type hints. But:

- Adding type hints **changes the source** → Python sees the file changed → recompiles to a new `.pyc`
- A file **without** any type hints still gets compiled to `.pyc` — Python always creates bytecode
- `from __future__ import annotations` makes annotation strings shorter → slightly smaller bytecode, faster import

**To be precise:** Python **always** creates `__pycache__` bytecode for any `.py` file that gets imported. Type hints do not control whether bytecode is created — only whether the source file is considered changed (triggering recompilation). If you delete `__pycache__`, Python recreates it on next import with or without type hints.

```python
# Without from __future__ import annotations:
# list[User] is evaluated at import — creates User object reference in bytecode

# With from __future__ import annotations:
# list[User] is stored as the string 'list[User]' — cheaper, forward references work
```

**Summary:**

| Concern | Impact |
|---|---|
| Runtime type enforcement | None — Python ignores annotations at runtime |
| Per-call overhead | None |
| Import time | Tiny — annotations parsed once |
| `__pycache__` creation | Always happens — type hints don't control it |
| Changing type hints | Triggers recompile of `.pyc` (source changed) |
| `from __future__ import annotations` | Slightly faster import, enables forward refs |

---
