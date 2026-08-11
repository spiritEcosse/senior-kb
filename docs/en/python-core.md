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

Alternative: `@contextlib.contextmanager` with `yield`.

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

`int`, `str`, `list[int]`, `dict[str, Any]`, `Optional[str]` = `str | None`
`TypeVar`, `Generic[T]` — generic types
`Callable[[int, str], bool]`
`Protocol` — defines a structural interface
Checkers: `mypy`, `pyright`

---

### functools

`@lru_cache(maxsize=128)` — memoisation with LRU eviction; pure functions only
`@cache` — unbounded cache (Python 3.9+)
`partial(func, *args)` — fix some arguments, return a new callable
`reduce(func, iterable)` — left fold

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
A metaclass controls class creation. It defines `__new__(cls, name, bases, attrs)`.

```python
class MyMeta(type):
    def __new__(cls, name, bases, attrs):
        attrs['custom_attr'] = 42
        return super().__new__(cls, name, bases, attrs)

class MyClass(metaclass=MyMeta):
    pass

print(MyClass.custom_attr)  # 42
```

Use cases: ORM (Django Models), API frameworks, class validation.

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
