## 13. Python vs C++ Performance

Python is slower because of:
1. Bytecode interpretation (CPython) vs machine code (C++)
2. Dynamic typing: type checked at runtime
3. GIL: limits parallelism
4. Reference counting and GC overhead
5. Everything is a heap object

How to speed it up: Cython, Numba (JIT), NumPy, PyPy, multiprocessing, asyncio, C/Rust extensions (cffi, pyo3).
Profile first: `cProfile`, `py-spy`, `line_profiler` — don't guess the bottleneck.

---
