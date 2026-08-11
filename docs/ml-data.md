# ML и данные: NumPy и Pandas

---

## NumPy

### Что такое NumPy и почему он быстрый?

**Краткий ответ:**
NumPy хранит данные в непрерывных C-массивах (`ndarray`) — фиксированный тип, без overhead Python-объектов. Операции выполняются в скомпилированном C/Fortran-коде, часто с освобождением GIL. Python `for`-цикл по списку обращается к Python-объекту на каждой итерации; NumPy-операция работает напрямую с сырой памятью.

```python
import numpy as np

# Python list — каждый элемент — объект кучи с тегом типа, refcount, указателем
lst = [1, 2, 3, 4, 5]

# NumPy array — непрерывный блок значений int64 в памяти
arr = np.array([1, 2, 3, 4, 5], dtype=np.int64)

print(arr.dtype)    # int64
print(arr.nbytes)   # 40 байт (5 × 8)
print(arr.itemsize) # 8
```

**Почему векторизация быстрее цикла:**

```python
import time

arr = np.arange(10_000_000)

# Медленно: Python-цикл — 10M обращений к Python-объектам
start = time.perf_counter()
result = [x * 2 for x in arr]
print(f"loop:  {time.perf_counter() - start:.3f}s")   # ~1.2s

# Быстро: векторизация — выполняется в C
start = time.perf_counter()
result = arr * 2
print(f"numpy: {time.perf_counter() - start:.3f}s")   # ~0.01s
```

---

### ndarray: оси, shape, reshape

```python
arr = np.array([[1, 2, 3],
                [4, 5, 6]])

print(arr.shape)   # (2, 3) — 2 строки, 3 столбца
print(arr.ndim)    # 2
print(arr.size)    # 6

# axis=0 → операция вдоль строк (схлопывает строки → одно значение на столбец)
print(arr.sum(axis=0))  # [5 7 9]

# axis=1 → операция вдоль столбцов (схлопывает столбцы → одно значение на строку)
print(arr.sum(axis=1))  # [6 15]

# reshape — без копирования если возможно (view тех же данных)
flat = arr.reshape(-1)      # [1 2 3 4 5 6]
col  = arr.reshape(3, 2)    # 3 строки, 2 столбца
auto = arr.reshape(6, -1)   # NumPy выводит размер -1 измерения
```

---

### Broadcasting

Правила применяются когда формы не совпадают — NumPy «растягивает» измерения размером 1.

```python
a = np.array([[1], [2], [3]])  # shape (3, 1)
b = np.array([10, 20, 30])     # shape (3,) → воспринимается как (1, 3)

print(a + b)
# [[11 21 31]
#  [12 22 32]
#  [13 23 33]]
```

**Правила broadcasting (выравнивание форм справа, сравнение поэлементно):**
1. Если количество измерений отличается — дополнить меньшую форму единицами слева
2. Измерения должны быть равны ИЛИ одно из них равно 1
3. Форма результата = максимум по каждому измерению

```python
# Частая ошибка:
a = np.ones((3, 4))
b = np.ones((4, 3))
# a + b → ERROR: shapes not broadcastable

# Исправление: транспонирование
a + b.T   # (3,4) + (3,4) ✓
```

---

### View vs copy

```python
arr = np.arange(10)

# View — разделяет память
view = arr[2:5]
view[0] = 99
print(arr)   # [ 0  1 99  3  4  5  6  7  8  9] — оригинал изменился!

# Copy — независимый
copy = arr[2:5].copy()
copy[0] = 0
print(arr)   # не изменился

# Проверка
print(np.shares_memory(arr, view))   # True
print(np.shares_memory(arr, copy))   # False
```

!!! warning
    Fancy indexing (`arr[[0, 1, 2]]`) всегда возвращает копию. Срезы возвращают view.

---

### Основные операции

```python
arr = np.array([3, 1, 4, 1, 5, 9, 2, 6])

# Агрегации
arr.mean(), arr.std(), arr.min(), arr.max()
np.median(arr)
np.percentile(arr, 75)

# Сортировка
np.sort(arr)           # возвращает отсортированную копию
arr.sort()             # in-place
np.argsort(arr)        # индексы, которые бы отсортировали arr

# Булева маска
mask = arr > 3
arr[mask]              # [4 5 9 6]
arr[arr > 3] *= -1     # in-place по маскированным элементам

# Линейная алгебра
A = np.random.randn(3, 3)
np.dot(A, A.T)         # матричное умножение
A @ A.T                # то же, синтаксис Python 3.5+
np.linalg.inv(A)
np.linalg.eig(A)
```

---

### dtype и оптимизация памяти

```python
# Тип по умолчанию float64 (8 байт) — часто избыточно
arr = np.ones(1_000_000)
print(arr.nbytes)   # 8 МБ

# float32 достаточно для большинства ML-задач
arr32 = arr.astype(np.float32)
print(arr32.nbytes) # 4 МБ — вдвое меньше

# Целочисленные типы
np.int8   # -128..127        1 байт
np.int16  # -32768..32767    2 байта
np.int32  # ±2B              4 байта
np.int64  # ±9e18            8 байт  (по умолчанию)

# Проверить перед понижением типа
print(arr.max(), arr.min())  # убедиться, что значения помещаются
```

---

## Pandas

### Series и DataFrame: внутреннее устройство

**Краткий ответ:**
`Series` — одномерный массив с метками, основанный на NumPy (или Arrow/ExtensionArray в новых версиях). `DataFrame` — словарь `Series`, разделяющих один индекс; колоночное хранилище.

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

# [] — доступ по столбцу (только метка)
df['a']          # Series столбца 'a'

# .loc — по метке (метка строки, метка столбца)
df.loc[10]           # строка с индексной меткой 10
df.loc[10, 'a']      # значение по метке 10, столбец 'a'
df.loc[10:20, 'a']   # строки 10..20 включительно (!)

# .iloc — по целочисленной позиции
df.iloc[0]           # первая строка
df.iloc[0, 1]        # строка 0, столбец 1 → 4
df.iloc[0:2]         # первые 2 строки (конец не включается, как в Python)
```

!!! warning
    Срез `.loc` включает конец; срез `.iloc` — не включает. Как срезы Python-списков.

---

### apply vs векторизованные операции

```python
df = pd.DataFrame({'price': [10.5, 20.0, 30.75], 'qty': [3, 1, 2]})

# Плохо — apply выполняет Python-цикл под капотом
df['total'] = df.apply(lambda row: row['price'] * row['qty'], axis=1)

# Хорошо — векторизованно: работает с целыми столбцами в C
df['total'] = df['price'] * df['qty']

# Для строковых операций — str-аксессор (векторизованный):
df['name'] = df['name'].str.upper()
```

**Правило:** `apply` — только когда нет векторизованного аналога. Он в ~10–100× медленнее.

---

### groupby

```python
df = pd.DataFrame({
    'dept':   ['eng', 'eng', 'hr', 'hr', 'eng'],
    'salary': [90, 95, 60, 65, 85],
    'level':  ['senior', 'senior', 'junior', 'senior', 'junior'],
})

# Базовая агрегация
df.groupby('dept')['salary'].mean()
# dept
# eng    90.0
# hr     62.5

# Несколько агрегаций
df.groupby('dept')['salary'].agg(['mean', 'max', 'count'])

# Несколько ключей
df.groupby(['dept', 'level'])['salary'].mean()

# transform — транслировать результат обратно на исходный индекс (нормализация)
df['salary_norm'] = df.groupby('dept')['salary'].transform(
    lambda x: (x - x.mean()) / x.std()
)
```

---

### merge (типы соединений)

```python
users  = pd.DataFrame({'id': [1, 2, 3], 'name': ['A', 'B', 'C']})
orders = pd.DataFrame({'user_id': [1, 1, 2], 'amount': [50, 30, 80]})

# inner — только совпадающие строки (по умолчанию)
pd.merge(users, orders, left_on='id', right_on='user_id')

# left — все строки из левого, NaN где нет совпадения
pd.merge(users, orders, left_on='id', right_on='user_id', how='left')

# right / outer — аналогично
```

| SQL | pandas `how=` |
|---|---|
| INNER JOIN | `'inner'` (по умолчанию) |
| LEFT JOIN | `'left'` |
| RIGHT JOIN | `'right'` |
| FULL OUTER | `'outer'` |

---

### Пропущенные данные

```python
df = pd.DataFrame({'a': [1, None, 3], 'b': [4, 5, None]})

df.isnull()          # булева маска
df.isnull().sum()    # количество NaN по столбцам
df.dropna()          # удалить строки с любым NaN
df.dropna(subset=['a'])   # удалить только где 'a' NaN
df.fillna(0)         # заменить NaN на 0
df['a'].fillna(df['a'].median())  # заполнить медианой
```

---

### Оптимизация памяти

```python
df = pd.read_csv('large.csv')
print(df.memory_usage(deep=True).sum() / 1e6, 'МБ')

# Понизить тип чисел
df['age'] = pd.to_numeric(df['age'], downcast='integer')   # int64 → int8/16

# Категориальные для строк с малым числом уникальных значений
df['status'] = df['status'].astype('category')
# 'pending'/'done'/'failed' — 3 значения → хранит как int-коды + таблица поиска

# Чтение по чанкам для файлов, не помещающихся в RAM
for chunk in pd.read_csv('large.csv', chunksize=100_000):
    process(chunk)
```

---

### Частые вопросы на интервью

**В: Чем `copy()` отличается от view в pandas?**

```python
# Срез DataFrame — может быть view или copy (зависит от реализации)
sub = df[df['score'] > 80]
sub['score'] = 0   # SettingWithCopyWarning!

# Всегда явно:
sub = df[df['score'] > 80].copy()
sub['score'] = 0   # безопасно, изменяет только sub
```

**В: Как избежать SettingWithCopyWarning?**
Всегда вызывать `.copy()` после булевой индексации перед изменением.

**В: `concat` vs `merge`?**
- `concat` — складывает DataFrame вертикально (строки) или горизонтально (столбцы), без сопоставления ключей
- `merge` — SQL-style join по ключевым столбцам

```python
# concat: сложить строки
pd.concat([df1, df2], ignore_index=True)

# concat: сложить столбцы рядом
pd.concat([df1, df2], axis=1)
```

**В: Как обработать CSV, не помещающийся в RAM?**

```python
result = []
for chunk in pd.read_csv('huge.csv', chunksize=50_000):
    result.append(chunk[chunk['value'] > 0].groupby('category')['value'].sum())

final = pd.concat(result).groupby(level=0).sum()
```

Или **Dask** — drop-in pandas API, разбивающий данные по ядрам/дискам:

```python
import dask.dataframe as dd
df = dd.read_csv('huge.csv')
result = df[df['value'] > 0].groupby('category')['value'].sum().compute()
```

---
