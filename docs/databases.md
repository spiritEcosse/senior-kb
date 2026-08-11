## 4. Базы данных

### ACID

A — Atomicity: всё или ничего
C — Consistency: БД переходит из одного корректного состояния в другое
I — Isolation: параллельные транзакции не мешают друг другу
D — Durability: после коммита данные сохранены навсегда (WAL, диск)

### Уровни изоляции транзакций

1. Read Uncommitted — возможно «грязное чтение»
2. Read Committed — только зафиксированные данные (по умолчанию: PostgreSQL, Oracle)
3. Repeatable Read — прочитанные данные не меняются до конца транзакции; фантомное чтение возможно
4. Serializable — максимальная изоляция; нет фантомного чтения

### Транзакция и ROLLBACK

Транзакция — логическая единица работы; атомарна.
ROLLBACK — отмена всех изменений с начала транзакции (или с SAVEPOINT).

### Виды индексов

1. Кластеризованный — физически упорядочивает строки; один на таблицу; листья = реальные данные
2. Некластеризованный — листья = ключ + указатель; их может быть много
3. Составной — несколько столбцов; важен порядок для prefix-запросов
4. Уникальный — гарантирует уникальность
5. Покрывающий — все нужные столбцы в индексе; без обращения к таблице

🔗 [https://habr.com/ru/post/247373/](https://habr.com/ru/post/247373/)

### N+1 проблема в ORM

```python
books = Book.objects.all()           # 1 запрос
for book in books:
    print(book.author.name)          # N запросов (по одному на книгу)
```

Решение в Django:
- `select_related` — SQL JOIN для FK и OneToOne (один запрос)
- `prefetch_related` — отдельный запрос + Python-объединение для M2M и обратных FK

```python
# Правильно:
books = Book.objects.select_related('author').all()
```

### annotate vs aggregate

```python
# aggregate — одно значение на весь queryset
Order.objects.aggregate(total=Sum('amount'))
# → {'total': 9500}

# annotate — значение на каждую строку
Order.objects.values('user_id').annotate(total=Sum('amount'))
# → [{'user_id': 1, 'total': 500}, {'user_id': 2, 'total': 300}, ...]
```

### SQL Window Functions

```sql
-- RANK() — ранг с пропусками при совпадении
SELECT name, salary,
       RANK() OVER (PARTITION BY dept ORDER BY salary DESC) as rank
FROM employees;

-- Running SUM — нарастающий итог
SELECT date, amount,
       SUM(amount) OVER (ORDER BY date) as running_total
FROM orders;
```

### Индексы PostgreSQL: составной и частичный

```sql
-- Составной: порядок столбцов важен для prefix-запросов
CREATE INDEX idx_user_created ON orders(user_id, created_at);
-- ✓ WHERE user_id = 1
-- ✓ WHERE user_id = 1 AND created_at > '2024-01-01'
-- ✗ WHERE created_at > '2024-01-01'  (без user_id — не использует индекс)

-- Частичный: индексирует только подмножество строк
CREATE INDEX idx_pending_orders ON orders(created_at)
WHERE status = 'pending';
-- Меньше размер, быстрее update, полезен при высокой избирательности
```

### EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;
```

Искать:
- `Seq Scan` на большой таблице → нет индекса
- Завышенные `rows` в оценке → устаревшая статистика → запустить `ANALYZE`
- `Nested Loop` с большим числом итераций → возможно нужен другой тип JOIN

### CAP-теорема

Нельзя одновременно:
- C — Consistency: все узлы видят одни данные
- A — Availability: каждый запрос получает ответ
- P — Partition Tolerance: система работает при разрыве сети

P почти всегда обязательна → выбирают CP или AP.
CP: HBase, Zookeeper. AP: Cassandra, CouchDB.

### UNION vs UNION ALL

`UNION` — удаляет дубликаты (дороже: нужна сортировка/хеширование).
`UNION ALL` — быстрее; дубликаты остаются.

### Масштабирование реляционной БД

- Вертикальное: более мощный сервер
- Горизонтальное: read replicas, шардирование
- Connection pooling: PgBouncer
- Кэширование: Redis (cache-aside)
- Денормализация для read-heavy нагрузок
- Оптимизация запросов: индексы, `EXPLAIN ANALYZE`, covering indexes, избегать `SELECT *`

---
