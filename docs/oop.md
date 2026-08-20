## 2. OOP

### Multiple Inheritance and MRO

Name conflicts — MRO: top-down, left-to-right. Algorithm: C3 Linearisation.
`MyClass.mro()` or `MyClass.__mro__` — inspect the order.
`super()` follows MRO; always use it instead of the parent's name directly.

**Example 1 — the classic diamond:**

```python
class A:
    def greet(self): return "A"

class B(A):
    def greet(self): return "B"

class C(A):
    def greet(self): return "C"

class D(B, C):
    pass

D().greet()       # "B" — MRO picks the first class (left-to-right) that defines it
D.mro()
# [<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>]
```

C3 guarantees: a subclass always comes before its parents, and the left-to-right order of `bases` in `class D(B, C)` is preserved — `B` before `C` because `B` was listed first, even though both inherit from `A`. Note `A` appears only *once*, after both `B` and `C` — C3 merges the diamond instead of visiting `A` twice.

**Example 2 — cooperative `super()` chains (why you always call `super().__init__()`):**

```python
class Base:
    def __init__(self):
        print("Base.__init__")

class Logging(Base):
    def __init__(self):
        print("Logging.__init__")
        super().__init__()      # NOT Base.__init__() — follows the MRO, not the literal parent

class Caching(Base):
    def __init__(self):
        print("Caching.__init__")
        super().__init__()

class Service(Logging, Caching):
    def __init__(self):
        print("Service.__init__")
        super().__init__()

Service()
# Service.__init__
# Logging.__init__
# Caching.__init__   ← reached via super(), NOT because Logging calls it directly
# Base.__init__      ← called exactly once, not twice

Service.mro()
# [Service, Logging, Caching, Base, object]
```

Each `super().__init__()` call advances to the *next class in the MRO*, not to "my direct parent" — that's what lets `Logging.__init__`'s `super()` land on `Caching` rather than jumping straight to `Base`. This is the mechanism Django's class-based views and mixins rely on (`ListView(LoginRequiredMixin, ListView)` etc.) — every mixin in the chain must call `super()` or the chain breaks.

**Example 3 — an MRO that's impossible to resolve:**

```python
class A: pass
class B(A): pass

class C(A, B):     # A appears before B in bases, but B is a subclass of A —
    pass            # contradicts "subclass before parent"
# TypeError: Cannot create a consistent method resolution
# order (MRO) for bases A, B
```

C3 fails here because the requested base order (`A` then `B`) conflicts with the rule that a class must come before its own base (`B` before `A`, since `B(A)`) — there's no linearisation that satisfies both.

---

### SOLID

S — Single Responsibility: one class = one reason to change
O — Open/Closed: extend via inheritance/composition, don't modify
L — Liskov Substitution: a subclass is fully interchangeable with its parent
I — Interface Segregation: many narrow interfaces > one broad one
D — Dependency Inversion: both levels depend on abstractions

---

### Design Patterns

**Creational:**
- Singleton: one instance. In Python: module-level object, or `__new__`, or metaclass.
- Factory Method: delegate object creation to subclasses.
- Abstract Factory: create families of related objects.

**Structural:**
- Decorator: add behaviour by wrapping (Python `@` syntax).
- Proxy: control access to an object (cache, logging, authorisation).
- Adapter: adapt one interface to another.

**Behavioural:**
- Observer: publisher notifies subscribers on change (Django signals).
- Strategy: swap in a different algorithm at runtime.
- Command: encapsulate a request as an object (undo/redo).

---

### Dataclasses

Auto-generates `__init__`, `__repr__`, `__eq__` from annotations.

!!! warning
    `__hash__` is set to `None` by default (object is unhashable, like any mutable).

`__hash__` is only generated when `frozen=True`.
`@dataclass(frozen=True)` — immutable (hashable).
`@dataclass(slots=True)` (Python 3.10+) — memory savings.

---
