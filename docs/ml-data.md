# ML & Data: NumPy and Pandas

---

## NumPy

### What is NumPy and why is it fast?

**Short answer:**
NumPy stores data in contiguous C arrays (`ndarray`) — fixed type, no Python object overhead. Operations run in compiled C/Fortran code, often releasing the GIL. A Python `for` loop on a list touches a Python object per iteration; a NumPy vectorised op touches raw memory.

**In depth:**

```python
import numpy as np

# Python list — each element is a heap object with type tag, refcount, pointer
lst = [1, 2, 3, 4, 5]

# NumPy array — raw block of int64 values, contiguous in memory
arr = np.array([1, 2, 3, 4, 5], dtype=np.int64)

print(arr.dtype)    # int64
print(arr.nbytes)   # 40 bytes (5 × 8)
print(arr.itemsize) # 8
```

**Why vectorised > loop:**

```python
import time

arr = np.arange(10_000_000)

# Slow: Python loop — 10M Python object interactions
start = time.perf_counter()
result = [x * 2 for x in arr]
print(f"loop:  {time.perf_counter() - start:.3f}s")   # ~1.2s

# Fast: vectorised — runs in C
start = time.perf_counter()
result = arr * 2
print(f"numpy: {time.perf_counter() - start:.3f}s")   # ~0.01s
```

---

### ndarray: axes, shape, reshape

```python
arr = np.array([[1, 2, 3],
                [4, 5, 6]])

print(arr.shape)   # (2, 3) — 2 rows, 3 cols
print(arr.ndim)    # 2
print(arr.size)    # 6

# axis=0 → operation along rows (collapse rows → one value per column)
print(arr.sum(axis=0))  # [5 7 9]

# axis=1 → operation along columns (collapse cols → one value per row)
print(arr.sum(axis=1))  # [6 15]

# reshape — no copy if possible (view of same data)
flat = arr.reshape(-1)      # [1 2 3 4 5 6]
col  = arr.reshape(3, 2)    # 3 rows, 2 cols
auto = arr.reshape(6, -1)   # NumPy infers the -1 dimension
```

---

### Broadcasting

Rules applied when shapes don't match — NumPy "stretches" dimensions of size 1.

```python
a = np.array([[1], [2], [3]])  # shape (3, 1)
b = np.array([10, 20, 30])     # shape (3,) → treated as (1, 3)

print(a + b)
# [[11 21 31]
#  [12 22 32]
#  [13 23 33]]
```

**Broadcasting rules (right-align shapes, compare element-wise):**
1. If dims differ, prepend 1s to the shorter shape
2. Dimensions must be equal OR one of them must be 1
3. Output shape = max of each dimension

```python
# Common mistake:
a = np.ones((3, 4))
b = np.ones((4, 3))
# a + b → ERROR: shapes not broadcastable

# Fix: transpose
a + b.T   # (3,4) + (3,4) ✓
```

---

### Views vs copies

```python
arr = np.arange(10)

# View — shares memory
view = arr[2:5]
view[0] = 99
print(arr)   # [ 0  1 99  3  4  5  6  7  8  9] — original changed!

# Copy — independent
copy = arr[2:5].copy()
copy[0] = 0
print(arr)   # unchanged

# Check
print(np.shares_memory(arr, view))   # True
print(np.shares_memory(arr, copy))   # False
```

!!! warning
    Fancy indexing (`arr[[0, 1, 2]]`) always returns a copy. Slicing returns a view.

---

### Common operations

```python
arr = np.array([3, 1, 4, 1, 5, 9, 2, 6])

# Aggregations
arr.mean(), arr.std(), arr.min(), arr.max()
np.median(arr)
np.percentile(arr, 75)

# Sorting
np.sort(arr)           # returns sorted copy
arr.sort()             # in-place
np.argsort(arr)        # indices that would sort arr

# Boolean masking
mask = arr > 3
arr[mask]              # [4 5 9 6]
arr[arr > 3] *= -1     # in-place on masked elements

# Linear algebra
A = np.random.randn(3, 3)
np.dot(A, A.T)         # matrix multiply
A @ A.T                # same, Python 3.5+ syntax
np.linalg.inv(A)
np.linalg.eig(A)
```

---

### dtype and memory optimisation

```python
# Default dtype is float64 (8 bytes) — often overkill
arr = np.ones(1_000_000)
print(arr.nbytes)   # 8 MB

# float32 is enough for most ML
arr32 = arr.astype(np.float32)
print(arr32.nbytes) # 4 MB — half the memory

# Integer types
np.int8   # -128..127        1 byte
np.int16  # -32768..32767    2 bytes
np.int32  # ±2B              4 bytes
np.int64  # ±9e18            8 bytes  (default)

# Check before downcasting
print(arr.max(), arr.min())  # make sure values fit
```

---

## Pandas

### Series and DataFrame internals

**Short answer:**
A `Series` is a 1D labelled array backed by a NumPy array (or Arrow/ExtensionArray in newer versions). A `DataFrame` is a dict of `Series` sharing an index — column-oriented storage.

```python
import pandas as pd

s = pd.Series([10, 20, 30], index=['a', 'b', 'c'])
print(s['b'])    # 20
print(s.values)  # numpy array: [10 20 30]

df = pd.DataFrame({
    'name':  ['Alice', 'Bob', 'Charlie'],
    'score': [95, 82, 78],
    'grade': ['A', 'B', 'C'],
})
print(df.dtypes)
# name     object
# score     int64
# grade    object
```

---

### loc vs iloc vs []

```python
df = pd.DataFrame({'a': [1,2,3], 'b': [4,5,6]}, index=[10, 20, 30])

# [] — column access (label only)
df['a']          # Series for column 'a'

# .loc — label-based (row label, column label)
df.loc[10]           # row with index label 10
df.loc[10, 'a']      # value at label 10, column 'a'
df.loc[10:20, 'a']   # rows 10..20 inclusive (!)

# .iloc — integer position-based
df.iloc[0]           # first row
df.iloc[0, 1]        # row 0, col 1 → 4
df.iloc[0:2]         # first 2 rows (exclusive end, like Python slicing)
```

!!! warning
    `.loc` slice end is **inclusive**; `.iloc` slice end is **exclusive** — same as Python lists.

---

### apply vs vectorised operations

```python
df = pd.DataFrame({'price': [10.5, 20.0, 30.75], 'qty': [3, 1, 2]})

# Bad — apply runs Python loop under the hood
df['total'] = df.apply(lambda row: row['price'] * row['qty'], axis=1)

# Good — vectorised: operates on whole columns in C
df['total'] = df['price'] * df['qty']

# Slightly better than apply for element-wise string ops:
df['name'] = df['name'].str.upper()    # str accessor — vectorised
```

**Rule:** reach for `apply` only when no vectorised equivalent exists. It's ~10–100× slower.

---

### groupby

```python
df = pd.DataFrame({
    'dept':   ['eng', 'eng', 'hr', 'hr', 'eng'],
    'salary': [90, 95, 60, 65, 85],
    'level':  ['senior', 'senior', 'junior', 'senior', 'junior'],
})

# Basic aggregation
df.groupby('dept')['salary'].mean()
# dept
# eng    90.0
# hr     62.5

# Multiple aggregations
df.groupby('dept')['salary'].agg(['mean', 'max', 'count'])

# Multiple keys
df.groupby(['dept', 'level'])['salary'].mean()

# transform — broadcast result back to original index (useful for normalisation)
df['salary_norm'] = df.groupby('dept')['salary'].transform(
    lambda x: (x - x.mean()) / x.std()
)
```

---

### merge (join types)

```python
users  = pd.DataFrame({'id': [1, 2, 3], 'name': ['A', 'B', 'C']})
orders = pd.DataFrame({'user_id': [1, 1, 2], 'amount': [50, 30, 80]})

# inner — only matching rows (default)
pd.merge(users, orders, left_on='id', right_on='user_id')

# left — all rows from left, NaN where no match
pd.merge(users, orders, left_on='id', right_on='user_id', how='left')

# right / outer — analogous

# Join on index
pd.merge(users.set_index('id'), orders.set_index('user_id'),
         left_index=True, right_index=True)
```

| SQL | pandas `how=` |
|---|---|
| INNER JOIN | `'inner'` (default) |
| LEFT JOIN | `'left'` |
| RIGHT JOIN | `'right'` |
| FULL OUTER | `'outer'` |

---

### Missing data

```python
df = pd.DataFrame({'a': [1, None, 3], 'b': [4, 5, None]})

df.isnull()          # boolean mask
df.isnull().sum()    # count NaN per column
df.dropna()          # drop rows with any NaN
df.dropna(subset=['a'])   # drop only where 'a' is NaN
df.fillna(0)         # replace NaN with 0
df['a'].fillna(df['a'].median())  # fill with median
```

---

### Memory optimisation

```python
df = pd.read_csv('large.csv')
print(df.memory_usage(deep=True).sum() / 1e6, 'MB')

# Downcast numerics
df['age'] = pd.to_numeric(df['age'], downcast='integer')   # int64 → int8/16

# Categoricals for low-cardinality string columns
df['status'] = df['status'].astype('category')
# 'pending'/'done'/'failed' — 3 values → stores as int codes + lookup table

# Chunked reading for files too large to fit in memory
for chunk in pd.read_csv('large.csv', chunksize=100_000):
    process(chunk)
```

---

### Common interview questions

**Q: What's the difference between `copy()` and a view in pandas?**

```python
# Slice of DataFrame — may be a view or copy (implementation-defined)
sub = df[df['score'] > 80]
sub['score'] = 0   # SettingWithCopyWarning!

# Always explicit:
sub = df[df['score'] > 80].copy()
sub['score'] = 0   # safe, modifies only sub
```

**Q: How do you avoid SettingWithCopyWarning?**
Always call `.copy()` after boolean indexing before modifying.

**Q: `concat` vs `merge`?**
- `concat` — stack DataFrames vertically (rows) or horizontally (columns), no key matching
- `merge` — SQL-style join on key columns

```python
# concat: stack rows
pd.concat([df1, df2], ignore_index=True)

# concat: stack columns side by side
pd.concat([df1, df2], axis=1)
```

**Q: How to find and remove duplicates?**

```python
df.duplicated()                    # boolean mask
df.duplicated(subset=['name'])     # duplicates on specific columns
df.drop_duplicates()               # remove duplicate rows
df.drop_duplicates(subset=['name'], keep='last')
```

**Q: How would you process a CSV that doesn't fit in RAM?**

```python
result = []
for chunk in pd.read_csv('huge.csv', chunksize=50_000):
    # filter, aggregate per chunk
    result.append(chunk[chunk['value'] > 0].groupby('category')['value'].sum())

final = pd.concat(result).groupby(level=0).sum()
```

Or use **Dask** — drop-in pandas API that partitions data across cores/disks:

```python
import dask.dataframe as dd
df = dd.read_csv('huge.csv')
result = df[df['value'] > 0].groupby('category')['value'].sum().compute()
```

---

---

## Pandas — Advanced

### Querying by columns

```python
# Select specific columns
df[['name', 'age']]
df.loc[:, ['name', 'age']]       # label-based, explicit
df.iloc[:, [0, 2]]               # position-based

# Filter rows
df[df['age'] > 30]
df.query('age > 30 and city == "Madrid"')   # string DSL — readable, slightly faster on large frames

# Multiple conditions
df[(df['age'] > 30) & (df['salary'] < 80_000)]

# isin / between
df[df['city'].isin(['Madrid', 'Valencia'])]
df[df['age'].between(25, 40)]

# Add derived column without mutation
df = df.assign(senior=df['age'] >= 60)

# Select only numeric columns
df.select_dtypes(include='number')

# Drop columns
df.drop(columns=['col1', 'col2'])
```

---

### Reading huge files that don't fit in memory

#### `chunksize` — sequential streaming

```python
result = []
for chunk in pd.read_csv('huge.csv', chunksize=100_000):
    # Each chunk is a full DataFrame of 100k rows
    result.append(chunk[chunk['value'] > 0].groupby('category')['value'].sum())

final = pd.concat(result).groupby(level=0).sum()
```

!!! note "Does pandas pre-scan the file?"
    **No.** `chunksize` returns a `TextFileReader` iterator. Pandas reads **sequentially**, N rows at a time, and discards each chunk after you process it. There is **no forecast, no pre-scan, no schema inference** beyond the header row. If column types vary mid-file (e.g. a string appears in what looked like an int column), pandas will raise mid-iteration. Use `dtype=` explicitly to avoid surprises:

```python
for chunk in pd.read_csv('huge.csv', chunksize=100_000,
                          usecols=['user_id', 'amount', 'status'],
                          dtype={'user_id': 'int32', 'amount': 'float32', 'status': 'category'}):
    process(chunk)
```

#### Read only specific columns

```python
# Read only what you need — huge memory saving
df = pd.read_csv('huge.csv', usecols=['user_id', 'created_at', 'amount'])

# With chunking
for chunk in pd.read_csv('huge.csv',
                          usecols=['user_id', 'amount'],
                          chunksize=50_000):
    process(chunk)
```

#### Dask — parallel, lazy, out-of-core

Dask is **not** a wrapper around `pd.read_csv` with `chunksize`. It builds a **task graph** — a DAG of operations that are evaluated lazily only when you call `.compute()`. Dask partitions the file into chunks and can run them in parallel across CPU cores or even a distributed cluster.

```python
import dask.dataframe as dd

# Reads only the metadata + schema — no data loaded yet
df = dd.read_csv('huge.csv')           # or 'data/*.csv' for multiple files
print(df.dtypes)                       # available immediately
print(df.npartitions)                  # number of chunks Dask will use

# Build a lazy computation graph — nothing runs yet
result = (
    df[df['amount'] > 0]
      .groupby('category')['amount']
      .sum()
)

# .compute() executes the whole graph, returns a pandas Series
final = result.compute()
```

**How Dask differs from `chunksize`:**

| | `chunksize` | Dask |
|---|---|---|
| Execution | Sequential, one chunk at a time | Parallel across cores/cluster |
| API | You write the loop | Drop-in pandas API |
| Lazy evaluation | No | Yes — builds DAG first |
| Memory | One chunk in RAM | Only active partitions |
| Multi-file | Manual glob + concat | Native: `dd.read_csv('data/*.csv')` |
| Distributed | No | Yes (Dask Distributed) |
| `.compute()` needed | No | Yes — to get a pandas result |

```python
# Dask with specific columns and types
df = dd.read_csv('huge.csv',
                  usecols=['user_id', 'amount'],
                  dtype={'user_id': 'int32', 'amount': 'float32'})

# Works on multiple files transparently
df = dd.read_csv('data/2024-*.csv')
```

**When to use Dask vs chunksize:**
- `chunksize` — simple pipelines, single-pass aggregation, low complexity
- Dask — complex multi-step transformations, joins across partitions, multi-core speedup, or truly distributed workloads

---

### Pandas data types

| pandas dtype | Backed by | When to use |
|---|---|---|
| `int64` | NumPy | Default integer |
| `int32`, `int16`, `int8` | NumPy | Downcast to save memory |
| `float64` | NumPy | Default float |
| `float32` | NumPy | ML features, saves memory |
| `object` | Python objects | Strings (legacy default) |
| `string` | Arrow/StringDtype | Preferred for strings in pandas 2.x |
| `category` | int codes + lookup | Low-cardinality strings (status, gender) |
| `bool` | NumPy | Boolean flags |
| `datetime64[ns]` | NumPy | Timestamps |
| `timedelta64[ns]` | NumPy | Durations |
| `Int64` (nullable) | pandas ExtensionArray | Integer with NaN support |
| `Float64` (nullable) | pandas ExtensionArray | Float with explicit NA |

```python
# Check types
df.dtypes
df.memory_usage(deep=True)

# Downcast to save memory
df['age']    = pd.to_numeric(df['age'],    downcast='integer')   # int64 → int8
df['price']  = pd.to_numeric(df['price'],  downcast='float')     # float64 → float32
df['status'] = df['status'].astype('category')

# Explicit dtype on read
df = pd.read_csv('data.csv', dtype={
    'user_id': 'int32',
    'amount':  'float32',
    'status':  'category',
    'active':  'bool',
})
```

---

### Indexes in pandas

```python
# Default RangeIndex (0, 1, 2, ...) — free, no memory cost
df = pd.DataFrame({'name': ['Alice', 'Bob'], 'score': [95, 82]})

# Set a column as index — fast label-based lookups with .loc
df = df.set_index('name')
df.loc['Alice']   # O(1) hash lookup

# Reset index — turn it back into a column
df = df.reset_index()

# MultiIndex — hierarchical index
df = df.set_index(['dept', 'level'])
df.loc[('eng', 'senior')]

# DatetimeIndex — enables time-based slicing
df.index = pd.to_datetime(df['date'])
df.loc['2024-01']          # all rows in January 2024
df.loc['2024-01':'2024-03']

# sort_index — required for slicing to work correctly on non-monotonic indexes
df = df.sort_index()
```

**When does a pandas index actually speed things up?**
- `.loc[]` lookups on a set index are O(1) for unique indexes
- Time-series slicing on a `DatetimeIndex`
- `merge` / `join` on index columns skips a sort step
- `groupby` on an indexed column is marginally faster

