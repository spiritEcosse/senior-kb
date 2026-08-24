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

### PyPy vs CPython

| | CPython | PyPy |
|---|---|---|
| Execution model | Bytecode interpreted directly (+ optional JIT since 3.13, see below) | Tracing JIT — detects hot loops, compiles them to machine code at runtime |
| Reference implementation | Yes — the "standard" Python, what `python3` almost always is | Alternative implementation, itself written in RPython |
| Speed | Baseline | 3–10× faster on long-running, loop-heavy pure-Python code |
| Startup time | Fast (~10s of ms) | Slower — JIT warm-up cost before it gets fast |
| Memory usage | Lower baseline | Higher — JIT-compiled code and trace data cost memory |
| C extension compatibility | Native — CPython's C API is the reference | Partial, via `cpyext` compatibility shim; C extensions using CPython internals directly (rather than the stable ABI) may not work or run slower than native |
| GIL | Yes | Yes (PyPy has its own GIL too — it doesn't remove the GIL, it's a separate implementation of the same single-threaded-bytecode-execution constraint) |
| Best for | Anything using NumPy/pandas/Django/etc. — the ecosystem is built against CPython | Long-running, CPU-bound pure-Python programs with hot loops and few C-extension dependencies |

**Why the speedup isn't universal:** the JIT only pays off once a loop has run enough iterations to be flagged "hot" and get compiled — short scripts or code where every line only runs once or twice stay interpreter-speed, minus the added warm-up cost. It's also a genuinely separate interpreter — bugs, edge-case semantics, and library support can differ from CPython, which is why most production deployments default to CPython unless a specific hot loop justifies switching.

**In short:** PyPy is a drop-in *interpreter* replacement (same language, different execution engine) — not a compiler like Cython, and not a library like NumPy. Reach for it when profiling shows the bottleneck is pure-Python loops CPython can't vectorise away, and the app doesn't lean hard on C-extension internals.

---

### CPython's own JIT (PEP 744)

Separate from PyPy's tracing JIT and Numba's LLVM JIT — this is a JIT built directly into CPython itself. Introduced experimentally (opt-in build flag) starting with Python 3.13, continuing to mature into 3.14. Uses a "copy-and-patch" approach: at build time, template machine code is generated for each bytecode instruction; at runtime, hot code paths get those templates stitched together and specialized, without a separate compilation pass like Numba/LLVM.

- Optimizes *while the program runs* (hot loops get compiled after they've executed enough times), not ahead-of-time.
- Distinct from `@lru_cache`-style caching or the "specializing adaptive interpreter" work from 3.11+ (PEP 659), which speeds up the bytecode interpreter itself without generating machine code — the JIT is the next step past that.
- Not a drop-in replacement for Numba/PyPy on tight numerical loops yet — those remain the better choice when raw numeric throughput is the goal.

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
