# Numeric & Primitive Data Types

Python's own types don't have fixed widths — the "how many bytes / what range" question only becomes concrete when you cross into C-level territory: `struct`, `ctypes`, `array`, NumPy dtypes, or network protocols. This page is the reference table for that.

## Signed vs unsigned integers

Range for an N-bit integer:
- **Unsigned**: `0` to `2^N − 1`
- **Signed** (two's complement): `−2^(N−1)` to `2^(N−1) − 1`

| Width | Unsigned range | Signed range |
|---|---|---|
| 1 byte (8-bit) | 0 – 255 | −128 – 127 |
| 2 bytes (16-bit) | 0 – 65,535 | −32,768 – 32,767 |
| 4 bytes (32-bit) | 0 – 4,294,967,295 | −2,147,483,648 – 2,147,483,647 |
| 8 bytes (64-bit) | 0 – 18,446,744,073,709,551,615 | −9,223,372,036,854,775,808 – 9,223,372,036,854,775,807 |

Why signed range is asymmetric: one bit is the sign, but two's complement has only one representation of zero (no `+0`/`−0`), so the extra negative value (`−2^(N-1)`) has no positive counterpart.

```python
import ctypes

ctypes.c_uint8(-1).value    # 255 — wraps around (unsigned)
ctypes.c_int8(200).value    # -56 — overflow (signed 8-bit max is 127)
```

---

## Python's own `int`

CPython `int` is **arbitrary-precision** — no fixed width, no overflow, grows as needed (stored as an array of 30-bit "digits" internally on 64-bit builds). `sys.maxsize` (`2^63 − 1` on 64-bit) is the max size of a container/index, not a limit on integer values.

```python
2 ** 1000          # works fine, no OverflowError — Python ints don't overflow
sys.getsizeof(0)   # 28 bytes — even zero has object header overhead
sys.getsizeof(1)   # 28 bytes
sys.getsizeof(2**64)  # bigger — needs more "digit" slots
```

Small integers `-5..256` are cached singletons (`is` comparison works by accident for them — don't rely on it).

---

## `bool`

`bool` is a subclass of `int`: `True == 1`, `False == 0`, `isinstance(True, int)` is `True`. At the Python object level it's a full heap object (28 bytes, same as any `int`) — there's no 1-bit or 1-byte "packed" bool in pure Python. NumPy's `np.bool_` is the actual 1-byte version.

```python
True + True        # 2 — bool participates in int arithmetic
isinstance(True, int)  # True
```

---

## Floats

| Type | Bits | Approx. range | Precision |
|---|---|---|---|
| `float32` (C `float`) | 32 | ~±3.4e38 | ~7 decimal digits |
| `float64` (C `double`, Python `float`) | 64 | ~±1.8e308 | ~15–17 decimal digits |

Python's built-in `float` is always a C `double` (64-bit) — there's no native `float32` in the language; you get it via `struct`, `array('f', ...)`, or `numpy.float32`.

---

## Strings

`str` is a sequence of Unicode code points, not bytes — length is "how many characters," not "how many bytes." CPython uses **PEP 393 flexible representation**: each string is stored at the narrowest fixed width that fits its widest character —

| Storage width | Used when the string contains |
|---|---|
| 1 byte/char (Latin-1) | only code points ≤ 0xFF |
| 2 bytes/char (UCS-2) | max code point ≤ 0xFFFF |
| 4 bytes/char (UCS-4) | any code point above 0xFFFF (e.g. emoji) |

```python
sys.getsizeof("hello")   # ASCII-only → 1 byte/char storage
sys.getsizeof("héllo")   # still ≤ 0xFF → 1 byte/char
sys.getsizeof("hello🎉")  # emoji forces 4 bytes/char for the WHOLE string
```

`len("🎉")` is `1` (one code point) even though it's encoded as 4 bytes in UTF-8 and 4 in UTF-32 — encoding (`bytes`) and code-point count (`str` length) are different questions.

---

## `struct` / `ctypes` format cheat sheet

```python
import struct

struct.pack('B', 255)     # B = unsigned char (1 byte)  → b'\xff'
struct.pack('b', -128)    # b = signed char   (1 byte)
struct.pack('H', 65535)   # H = unsigned short (2 bytes)
struct.pack('h', -32768)  # h = signed short   (2 bytes)
struct.pack('I', 4294967295)  # I = unsigned int  (4 bytes)
struct.pack('i', -2147483648) # i = signed int    (4 bytes)
struct.pack('Q', 2**64 - 1)   # Q = unsigned long long (8 bytes)
struct.pack('q', -2**63)      # q = signed long long   (8 bytes)
struct.pack('f', 3.14)        # f = float  (4 bytes)
struct.pack('d', 3.14)        # d = double (8 bytes)
struct.pack('?', True)        # ? = bool   (1 byte)
```

Same letters map directly to `array.array(typecode, ...)` and to `ctypes.c_<name>` (`c_uint8`, `c_int8`, `c_uint16`, `c_int32`, ...). NumPy uses its own dtype names but the same widths: `np.uint8`, `np.int8`, `np.uint16`, `np.int32`, `np.int64`, `np.float32`, `np.float64` — see [ML & Data](ml-data.md) for why fixed-width dtypes matter for memory/vectorisation.

---

## Common pitfalls

- Assuming Python `int` behaves like a fixed-width C `int` — it doesn't overflow, so bugs that would surface as wraparound in C/Java silently "work" in pure Python until you serialize into a fixed-width field (`struct`, protobuf `int32`, a DB `INTEGER` column) and it overflows there instead.
- Comparing floats with `==` — `0.1 + 0.2 == 0.3` is `False` (binary float64 can't represent 0.1 exactly); use `math.isclose()`.
- Assuming `len(s)` on a `str` tells you the byte size for network/storage purposes — encode first (`len(s.encode('utf-8'))`).
