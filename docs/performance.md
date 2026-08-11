## 13. Python vs C++ Performance

### Why Python is slower

| Factor | Python | C++ |
|---|---|---|
| Execution | Bytecode interpreted by CPython | Compiled to native machine code |
| Types | Checked at runtime (dynamic) | Checked at compile time (static) |
| Memory | Every value is a heap object with refcount | Stack/heap, value types, no GC overhead |
| Threads | GIL limits parallelism | True multi-core parallelism |
| Dispatch | Virtual dispatch on every attribute access | Inlined, devirtualised by compiler |

---

### Profiling first — never guess

```python
# cProfile — built-in, call-level stats
python -m cProfile -s cumulative my_script.py

import cProfile
cProfile.run('my_function()', sort='cumulative')

# line_profiler — line-by-line (install: pip install line-profiler)
@profile
def slow_function():
    ...
kernprof -l -v my_script.py

# memory_profiler — memory usage per line
@profile
def memory_heavy():
    ...
python -m memory_profiler my_script.py

# py-spy — sampling profiler, zero code changes, safe in production
py-spy record -o profile.svg --pid 12345
py-spy top --pid 12345          # live top-like view
```

---

### Common optimisations

**1. Vectorise with NumPy — avoid Python loops on numerical data:**

```python
import numpy as np

# Slow: Python loop
result = [x ** 2 for x in data]       # ~1.2s for 10M elements

# Fast: NumPy vectorised
result = np.array(data) ** 2          # ~0.01s
```

**2. Use built-ins and standard library — implemented in C:**

```python
# Slow
total = 0
for x in data:
    total += x

# Fast
total = sum(data)          # C loop internally
```

**3. Avoid repeated attribute lookups in tight loops:**

```python
# Slow — looks up `math.sqrt` on every iteration
import math
for x in data:
    math.sqrt(x)

# Fast — local reference
from math import sqrt
for x in data:
    sqrt(x)
```

**4. Use generators for large sequences:**

```python
# Builds entire list in memory
total = sum([x ** 2 for x in range(10_000_000)])

# Generator — one element at a time
total = sum(x ** 2 for x in range(10_000_000))
```

**5. String concatenation:**

```python
# Slow: creates a new string object each iteration
result = ""
for s in parts:
    result += s

# Fast: single allocation
result = "".join(parts)
```

---

### Speed-up tools

| Tool | Approach | Speedup |
|---|---|---|
| NumPy | C arrays + BLAS | 10–100× for numerical |
| Cython | Compile Python to C | 10–100× |
| Numba | JIT compile with LLVM | 10–200× for numerical |
| PyPy | Tracing JIT interpreter | 3–10× general Python |
| multiprocessing | Bypass GIL with processes | N× (N = cores) |
| asyncio | Concurrent I/O, single thread | High concurrency, not raw speed |
| C extension (cffi, ctypes) | Call C/Rust libraries | Near-native |
| pyo3 | Write Python extensions in Rust | Near-native |

---

### When Python speed is good enough

CPython is fast enough for:
- I/O-bound work (network, disk) — speed is the network, not Python
- Orchestration code — gluing fast libraries together
- Prototyping — correctness first, optimise later

Python is genuinely slow for:
- Tight numerical loops without NumPy
- CPU-bound parallelism (GIL)
- Real-time / latency-sensitive systems (use Rust/C++ there)

---
