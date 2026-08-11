## 13. Производительность Python vs C++

Python медленнее из-за:
1. Интерпретация байткода (CPython) vs машинный код (C++)
2. Динамическая типизация: тип проверяется в runtime
3. GIL: ограничивает параллелизм
4. Overhead от reference counting и GC
5. Всё — объект на куче

Как ускорить: Cython, Numba (JIT), NumPy, PyPy, multiprocessing, asyncio, C/Rust-расширения (cffi, pyo3).
Профилировать сначала: cProfile, py-spy, line_profiler — не угадывать узкое место.
