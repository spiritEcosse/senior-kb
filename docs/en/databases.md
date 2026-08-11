## 4. Databases

### ACID

A — Atomicity: all or nothing
C — Consistency: DB transitions from one valid state to another
I — Isolation: concurrent transactions don't interfere
D — Durability: after commit, data is persisted forever (WAL, disk)

### Transaction Isolation Levels

1. Read Uncommitted — dirty reads possible
2. Read Committed — only committed data visible (default: PostgreSQL, Oracle)
3. Repeatable Read — read data doesn't change until transaction ends; phantom reads possible
4. Serializable — maximum isolation; no phantom reads

### Transactions and ROLLBACK

A transaction is a logical unit of work; it is atomic.
`ROLLBACK` — undoes all changes since the transaction began (or since a `SAVEPOINT`).

### Index Types

1. Clustered — physically orders rows; one per table; leaf pages = actual data
2. Non-clustered — leaf pages = key + pointer; many allowed
3. Composite — multiple columns; column order matters for prefix queries
4. Unique — enforces uniqueness
5. Covering — all needed columns in the index; no table lookup required

🔗 [https://habr.com/ru/post/247373/](https://habr.com/ru/post/247373/)

### N+1 Problem in ORM

```python
books = Book.objects.all()       # 1 query
for book in books:
    print(book.author.name)      # N queries (one per book)
```

Solution in Django:
- `select_related` — SQL JOIN for FK and OneToOne (one query)
- `prefetch_related` — separate query + Python join for M2M and reverse FK

```python
# Correct:
books = Book.objects.select_related('author').all()
```

### annotate vs aggregate

```python
# aggregate — single value for the whole queryset
Order.objects.aggregate(total=Sum('amount'))
# → {'total': 9500}

# annotate — value per row
Order.objects.values('user_id').annotate(total=Sum('amount'))
# → [{'user_id': 1, 'total': 500}, {'user_id': 2, 'total': 300}, ...]
```

### SQL Window Functions

```sql
-- RANK() — rank with gaps on tie
SELECT name, salary,
       RANK() OVER (PARTITION BY dept ORDER BY salary DESC) as rank
FROM employees;

-- Running SUM — cumulative total
SELECT date, amount,
       SUM(amount) OVER (ORDER BY date) as running_total
FROM orders;
```

### PostgreSQL Indexes: Composite and Partial

```sql
-- Composite: column order matters for prefix queries
CREATE INDEX idx_user_created ON orders(user_id, created_at);
-- ✓ WHERE user_id = 1
-- ✓ WHERE user_id = 1 AND created_at > '2024-01-01'
-- ✗ WHERE created_at > '2024-01-01'  (without user_id — won't use index)

-- Partial: indexes only a subset of rows
CREATE INDEX idx_pending_orders ON orders(created_at)
WHERE status = 'pending';
-- Smaller size, faster updates, useful when high selectivity
```

### EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;
```

Look for:
- `Seq Scan` on a large table → missing index
- Overestimated `rows` → stale stats → run `ANALYZE`
- `Nested Loop` with large iteration count → may need a different JOIN type

### CAP Theorem

You cannot have all three simultaneously:
- C — Consistency: all nodes see the same data
- A — Availability: every request gets a response
- P — Partition Tolerance: system works despite network splits

P is almost always required → choose CP or AP.
CP: HBase, Zookeeper. AP: Cassandra, CouchDB.

### UNION vs UNION ALL

`UNION` — removes duplicates (more expensive: requires sort/hash).
`UNION ALL` — faster; duplicates are kept.

### Scaling a Relational DB

- Vertical: more powerful server
- Horizontal: read replicas, sharding
- Connection pooling: PgBouncer
- Caching: Redis (cache-aside)
- Denormalisation for read-heavy workloads
- Query optimisation: indexes, `EXPLAIN ANALYZE`, covering indexes, avoid `SELECT *`

---
