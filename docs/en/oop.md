## 2. OOP

### Multiple Inheritance and MRO

Name conflicts — MRO: top-down, left-to-right. Algorithm: C3 Linearisation.
`MyClass.mro()` or `MyClass.__mro__` — inspect the order.
`super()` follows MRO; always use it instead of the parent's name directly.

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
