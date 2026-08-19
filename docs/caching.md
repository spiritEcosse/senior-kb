## 6. Caching

### Caching Strategies

Cache-Aside (Lazy Loading): request → miss → DB → write to cache. Most common.
Write-Through: write to cache AND DB simultaneously. Always consistent; slower writes.
Write-Behind: write to cache; async flush to DB. Fast writes; risk of data loss.

---

### Thread-safe Cache with TTL

**The problem — race condition in cache:**

```
Thread 1: if url in self.cache     # miss → makes request
Thread 2: if url in self.cache     # miss → makes request (duplicate!)
Thread 1: self.cache[url] = data   # writes
Thread 2: self.cache[url] = data   # overwrites
```

Python's GIL protects simple operations but **not compound read-check-write sequences**.

---

**Fix 1 — threading.Lock + manual TTL:**

```python
import threading
from time import time

class APIClient:
    def __init__(self, base_url, cache_ttl=60):
        self.base_url = base_url
        self.cache_ttl = cache_ttl
        self.cache = {}
        self._cache_lock = threading.Lock()

    def _get_cached(self, url):
        with self._cache_lock:
            if url in self.cache:
                entry = self.cache[url]
                if entry['expires_at'] > time():
                    return entry['data']
                del self.cache[url]
        return None

    def get(self, endpoint, use_cache=True):
        url = self.base_url + endpoint
        if use_cache:
            cached = self._get_cached(url)
            if cached:
                return cached

        data = self.session.get(url).json()

        with self._cache_lock:
            self.cache[url] = {
                'data': data,
                'expires_at': time() + self.cache_ttl
            }
        return data
```

---

**Fix 2 — threading.local() (per-thread cache):**

```python
class APIClient:
    def __init__(self, ...):
        self._local = threading.local()

    @property
    def cache(self):
        if not hasattr(self._local, 'cache'):
            self._local.cache = {}
        return self._local.cache
```

Each thread gets its **own cache** — no sharing, no locking needed. Simpler but uses more memory.

---

**Fix 3 — cachetools.TTLCache + RLock (recommended):**

```python
from cachetools import TTLCache
from threading import RLock

class APIClient:
    def __init__(self, base_url):
        self.base_url = base_url
        self.cache = TTLCache(
            maxsize=1000,   # max 1000 entries
            ttl=60          # 60 second TTL
        )
        self._cache_lock = RLock()

    def get(self, endpoint, use_cache=True):
        url = self.base_url + endpoint
        with self._cache_lock:
            if use_cache and url in self.cache:
                return self.cache[url]

        data = self.session.get(url).json()

        with self._cache_lock:
            self.cache[url] = data
        return data
```

`TTLCache` handles TTL and max size automatically. `RLock` instead of `Lock` — re-entrant, safer in complex call chains.

---

### Cache Invalidation via Django Signals

```python
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.core.cache import cache

@receiver(post_save, sender=Product)
def invalidate_product_cache(sender, instance, **kwargs):
    cache.delete(f'product:{instance.pk}')
    cache.delete('product:list')
```

---

### Redis: Use Cases

- Session storage
- Rate limiting (`INCR` + `EXPIRE`)
- Distributed lock (`SET NX PX`)
- Pub/Sub
- Leaderboards (Sorted Sets)
- Task queues (List `LPUSH`/`BRPOP`, Redis Streams)
- Cache with TTL (cache-aside)

---

### Rate Limiting

**Fixed Window (simplest):**

```python
import redis
from time import time

def is_allowed_fixed(user_id: str, limit: int = 100) -> bool:
    r = redis.Redis()
    key = f"rate:{user_id}:{int(time() // 60)}"  # 1 minute window
    count = r.incr(key)
    if count == 1:
        r.expire(key, 60)
    return count <= limit
```

Problem: double burst at window boundary (100 at end + 100 at start of next window).

---

**Sliding Window (more precise):**

```python
def is_allowed_sliding(user_id: str, limit: int = 100) -> bool:
    r = redis.Redis()
    key = f"rate:sliding:{user_id}"
    now = time()
    window_start = now - 60

    pipe = r.pipeline()
    pipe.zremrangebyscore(key, 0, window_start)   # remove old entries
    pipe.zadd(key, {str(now): now})               # add current
    pipe.zcard(key)                                # count
    pipe.expire(key, 60)
    results = pipe.execute()

    return results[2] <= limit
```

Sorted Set: score = timestamp. Precise sliding window, but O(log N) and more memory.

---

**Token Bucket (flexible burst):**

Bucket holds tokens (max = capacity). Each request takes a token; tokens refill over time. Empty bucket → reject. Handles bursts more fairly than fixed window, which is why it's preferred for fintech.

Naive `GET` then `SET` from Python has a race condition between concurrent requests — wrap the read-check-write in a Lua script so it runs as a single atomic op on the Redis server:

```python
import redis
import functools

class TokenBucketLimiter:
    LUA_SCRIPT = """
    local tokens = tonumber(redis.call('GET', KEYS[1]) or ARGV[1])
    if tokens > 0 then
        redis.call('SET', KEYS[1], tokens - 1)
        redis.call('EXPIRE', KEYS[1], ARGV[2])
        return 1
    else
        return 0
    end
    """

    def __init__(self, redis_client: redis.Redis, capacity: int = 100, window: int = 60):
        self.redis = redis_client
        self.capacity = capacity
        self.window = window
        # register once — compiles the script server-side and caches its SHA;
        # subsequent calls send EVALSHA instead of the full script text
        self.script = self.redis.register_script(self.LUA_SCRIPT)

    def allow(self, key: str) -> bool:
        result = self.script(keys=[key], args=[self.capacity, self.window])
        return bool(result)


limiter = TokenBucketLimiter(redis.Redis(host="localhost", port=6379, db=0))


def rate_limited(limiter: TokenBucketLimiter, key_func):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            key = key_func(*args, **kwargs)
            if not limiter.allow(key):
                raise RateLimitExceeded(key)
            return fn(*args, **kwargs)
        return wrapper
    return decorator


@rate_limited(limiter, key_func=lambda user_id, *a, **kw: f"rate_limit:{user_id}")
def call_external_api(user_id: str, *args, **kwargs):
    ...
```

- Keying strategy: `rate_limit:{user_id}` (global) or `rate_limit:{api_name}:{user_id}` (per-endpoint).
- Production pitfall: `register_script` returns a `Script` object tied to the connection at creation time. With a connection pool and failover, some client wrappers need re-registration after a reconnect, or calls raise `NOSCRIPT`.

---

**Nginx — global IP-level protection:**

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
limit_req zone=api burst=20 nodelay;
```

Works before Django — zero application overhead. Limitation: per-IP, not per-user. Problems with NAT, CDNs.

---

**Choosing a strategy:**

| Scenario | Approach |
|---|---|
| General API protection | Fixed window |
| Financial / trading API | Sliding window |
| Very high traffic | Nginx / API Gateway |
| Flexible burst | Token bucket |

---
