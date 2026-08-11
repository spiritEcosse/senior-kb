## 6. Caching

### Caching Strategies

Cache-Aside (Lazy Loading): request → miss → DB → write to cache. Most common.
Write-Through: write to cache AND DB simultaneously. Always consistent; slower writes.
Write-Behind: write to cache; async flush to DB. Fast writes; risk of data loss.

### Redis: Use Cases

- Session storage
- Rate limiting (`INCR` + `EXPIRE`)
- Distributed lock (`SET NX PX`)
- Pub/Sub
- Leaderboards (Sorted Sets)
- Task queues (List `LPUSH`/`BRPOP`, Redis Streams)
- Cache with TTL (cache-aside)

---
