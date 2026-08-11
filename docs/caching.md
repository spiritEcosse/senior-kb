## 6. Кэширование

### Стратегии кэширования

Cache-Aside (Lazy Loading): запрос → промах → БД → записать в кэш. Самая распространённая.
Write-Through: запись в кэш И в БД одновременно. Всегда согласовано; медленнее запись.
Write-Behind: запись в кэш; асинхронный сброс в БД. Быстрая запись; риск потери данных.

---

### Thread-safe кэш с TTL

**Проблема — race condition в кэше:**

```
Поток 1: if url in self.cache     # нет → делает запрос
Поток 2: if url in self.cache     # нет → делает запрос (дублирует!)
Поток 1: self.cache[url] = data   # пишет
Поток 2: self.cache[url] = data   # перезаписывает
```

GIL защищает простые операции, но не составные `read-check-write`.

---

**Fix 1 — threading.Lock + ручной TTL:**

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

**Fix 2 — threading.local() (кэш per-thread):**

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

Каждый поток получает **свой кэш** — нет sharing, нет локов. Проще, но больше памяти.

---

**Fix 3 — cachetools.TTLCache + RLock (рекомендуется):**

```python
from cachetools import TTLCache
from threading import RLock

class APIClient:
    def __init__(self, base_url):
        self.base_url = base_url
        self.cache = TTLCache(
            maxsize=1000,   # макс. 1000 записей
            ttl=60          # TTL 60 секунд
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

`TTLCache` управляет TTL и размером автоматически. `RLock` вместо `Lock` — можно захватить повторно из того же потока (безопаснее в сложных сценариях).

---

### Инвалидация кэша через Django signals

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

### Redis: сценарии использования

- Хранение сессий
- Rate limiting (`INCR` + `EXPIRE`)
- Distributed lock (`SET NX PX`)
- Pub/Sub
- Лидерборды (Sorted Sets)
- Очереди задач (List `LPUSH`/`BRPOP`, Redis Streams)
- Кэш с TTL (cache-aside)

---

### Rate Limiting

**Fixed Window (простейший):**

```python
import redis
from time import time

def is_allowed_fixed(user_id: str, limit: int = 100) -> bool:
    r = redis.Redis()
    key = f"rate:{user_id}:{int(time() // 60)}"  # окно = 1 минута
    count = r.incr(key)
    if count == 1:
        r.expire(key, 60)
    return count <= limit
```

Проблема: двойной burst на границе окна (100 запросов в конце + 100 в начале следующего).

---

**Sliding Window (точнее):**

```python
def is_allowed_sliding(user_id: str, limit: int = 100) -> bool:
    r = redis.Redis()
    key = f"rate:sliding:{user_id}"
    now = time()
    window_start = now - 60

    pipe = r.pipeline()
    pipe.zremrangebyscore(key, 0, window_start)   # удалить старые
    pipe.zadd(key, {str(now): now})               # добавить текущий
    pipe.zcard(key)                                # посчитать
    pipe.expire(key, 60)
    results = pipe.execute()

    return results[2] <= limit
```

Sorted Set: score = timestamp. Точное скользящее окно, но O(log N) и больше памяти.

---

**Token Bucket (гибкий burst):**

```python
# Пользователь получает 100 токенов, пополнение 1.67/сек
# Burst разрешён до размера bucket
# Нет проблемы с границей окна
```

---

**Nginx — глобальная защита на уровне IP:**

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
limit_req zone=api burst=20 nodelay;
```

Работает до Django — нулевой overhead приложения. Ограничение: per-IP, не per-user. Проблемы с NAT, CDN.

---

**Выбор стратегии:**

| Сценарий | Подход |
|---|---|
| Общая защита API | Fixed window |
| Финансовые / trading API | Sliding window |
| Очень высокий трафик | Nginx / API Gateway |
| Гибкий burst | Token bucket |

---
