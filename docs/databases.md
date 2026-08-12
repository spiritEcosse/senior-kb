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

### Composite and Partial Indexes

#### Composite index

Supported by all major databases (PostgreSQL, MySQL/InnoDB, SQLite, Oracle, MSSQL). The prefix rule works the same everywhere.

```sql
CREATE INDEX idx_user_created ON orders(user_id, created_at);
-- ✓ WHERE user_id = 1
-- ✓ WHERE user_id = 1 AND created_at > '2024-01-01'
-- ✗ WHERE created_at > '2024-01-01'  (no prefix — index not used)
```

Column order is the key: the index is only usable when the query filters on a **prefix** of the indexed columns.

#### Partial index

Indexes only a subset of rows matching a `WHERE` condition — smaller size, faster writes, higher selectivity.

**Database support:**

| Database | Support |
|---|---|
| PostgreSQL | ✅ Full support |
| SQLite | ✅ Full support |
| MSSQL | ✅ Called "filtered index" |
| Oracle | ⚠️ Via function-based indexes only |
| MySQL / MariaDB | ❌ Not supported |

**PostgreSQL / SQLite / MSSQL:**

```sql
-- Only indexes pending orders — much smaller than a full index on status
CREATE INDEX idx_pending_orders ON orders(created_at)
WHERE status = 'pending';
```

**MySQL workaround — generated column + regular index:**

```sql
-- Step 1: add a generated column that is NULL when not interesting
ALTER TABLE orders
  ADD COLUMN is_pending TINYINT(1) GENERATED ALWAYS AS (
    IF(status = 'pending', 1, NULL)
  ) STORED;

-- Step 2: index only the non-NULL values (NULL is never indexed in MySQL)
CREATE INDEX idx_pending_orders ON orders(is_pending, created_at);

-- Query uses the index:
SELECT * FROM orders WHERE is_pending = 1 AND created_at > '2024-01-01';
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

---

## MongoDB

### Creating schemas (Validation)

MongoDB is schemaless by default, but you can enforce structure via **JSON Schema validation**:

```javascript
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "email", "created_at"],
      properties: {
        name:       { bsonType: "string",   description: "required string" },
        email:      { bsonType: "string",   pattern: "^.+@.+$" },
        age:        { bsonType: "int",      minimum: 0, maximum: 150 },
        created_at: { bsonType: "date" },
        role:       { enum: ["admin", "user", "moderator"] }
      }
    }
  },
  validationAction: "error"   // or "warn" — log but allow
})
```

In PyMongo / Motor:

```python
import pymongo

client = pymongo.MongoClient("mongodb://localhost:27017")
db = client["myapp"]

db.command("collMod", "users", validator={
    "$jsonSchema": {
        "bsonType": "object",
        "required": ["name", "email"],
        "properties": {
            "name":  {"bsonType": "string"},
            "email": {"bsonType": "string"},
        }
    }
})
```

---

### ObjectId

`ObjectId` is MongoDB's default `_id` type — a 12-byte value:

```
4 bytes  — Unix timestamp (seconds) → creation time built in
5 bytes  — random value (machine + process)
3 bytes  — incrementing counter
```

```python
from bson import ObjectId

oid = ObjectId()
print(oid)                    # 665f3a1b2c4e5f6a7b8c9d0e
print(oid.generation_time)    # datetime — free, no extra query needed

# Query by ObjectId
db.users.find_one({"_id": ObjectId("665f3a1b2c4e5f6a7b8c9d0e")})

# Convert string → ObjectId when receiving from API
user_id = ObjectId(request_id_string)
```

**Why ObjectId over auto-increment int?**
- Generated client-side — no DB round trip, no sequence contention
- Sharding-friendly — no central sequence
- Carries timestamp — no `created_at` column needed for rough ordering

---

### References between collections (populate pattern)

MongoDB doesn't have JOINs, but you can reference documents by `_id`:

```python
# users collection
{ "_id": ObjectId("aaa"), "name": "Alice" }

# posts collection — stores reference to user
{ "_id": ObjectId("bbb"), "title": "Hello", "author_id": ObjectId("aaa") }
```

**Manual lookup (two queries):**

```python
post = db.posts.find_one({"_id": post_id})
author = db.users.find_one({"_id": post["author_id"]})
```

**`$lookup` — server-side join (aggregation pipeline):**

```javascript
db.posts.aggregate([
  {
    $lookup: {
      from:         "users",
      localField:   "author_id",
      foreignField: "_id",
      as:           "author"
    }
  },
  { $unwind: "$author" }   // flatten array → single object
])
```

**Return list of users, details in another collection** — this is the recommended pattern:

```python
# Fast: return just the list (lightweight)
users = list(db.users.find({}, {"_id": 1, "name": 1, "email": 1}))

# Lazy: fetch full profile only when needed
def get_user_profile(user_id):
    return db.user_profiles.find_one({"user_id": user_id})
```

Or with `$lookup` to get summary + details in one aggregation:

```javascript
// Get users with their profile details joined
db.users.aggregate([
  {
    $lookup: {
      from:         "user_profiles",
      localField:   "_id",
      foreignField: "user_id",
      as:           "profile"
    }
  },
  { $project: { name: 1, email: 1, "profile.bio": 1, "profile.avatar": 1 } }
])
```

---

### Indexes in MongoDB

```javascript
// Single field
db.users.createIndex({ email: 1 })          // ascending
db.users.createIndex({ email: -1 })         // descending

// Unique index
db.users.createIndex({ email: 1 }, { unique: true })

// Compound index — column order matters (same prefix rule as SQL)
db.orders.createIndex({ user_id: 1, created_at: -1 })

// Partial index — index only matching documents
db.orders.createIndex(
  { created_at: 1 },
  { partialFilterExpression: { status: "pending" } }
)

// TTL index — auto-delete documents after N seconds
db.sessions.createIndex({ created_at: 1 }, { expireAfterSeconds: 86400 })

// Text index — full-text search
db.articles.createIndex({ title: "text", body: "text" })
db.articles.find({ $text: { $search: "mongodb indexes" } })
```

In Python (PyMongo):

```python
from pymongo import ASCENDING, DESCENDING, IndexModel

db.users.create_index([("email", ASCENDING)], unique=True)

db.orders.create_indexes([
    IndexModel([("user_id", ASCENDING), ("created_at", DESCENDING)]),
    IndexModel([("status", ASCENDING)], partialFilterExpression={"status": "pending"})
])

# List current indexes
list(db.users.list_indexes())
```

---

### Profiling queries / improving speed

```javascript
// 1. explain() — see query plan
db.orders.find({ user_id: "abc" }).explain("executionStats")
// Look for:
//   COLLSCAN → full collection scan → missing index
//   IXSCAN   → index used → good
//   totalDocsExamined >> nReturned → low selectivity

// 2. Enable the profiler
db.setProfilingLevel(1, { slowms: 100 })  // log queries > 100ms
db.setProfilingLevel(2)                   // log ALL queries (dev only)

// 3. Read profiler output
db.system.profile.find().sort({ ts: -1 }).limit(10).pretty()
// Key fields: millis, op, ns, keysExamined, docsExamined, nreturned

// 4. currentOp() — see running queries right now
db.currentOp({ "active": true, "secs_running": { $gt: 1 } })

// 5. killOp() — kill a slow query
db.killOp(<opid>)
```

**Common fixes after profiling:**

| Problem | Fix |
|---|---|
| `COLLSCAN` on filtered field | Add index on that field |
| High `docsExamined / nreturned` ratio | Add more selective index or compound index |
| Slow sort | Add index matching sort order |
| Returning full documents | Use projection `{field: 1}` |
| Repeated identical queries | Cache in Redis |

---
