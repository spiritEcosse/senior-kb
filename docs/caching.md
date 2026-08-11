## 6. Кэширование

### Стратегии кэширования

Cache-Aside (Lazy Loading): запрос → промах → БД → записать в кэш. Самая распространённая.
Write-Through: запись в кэш И в БД одновременно. Всегда согласовано; медленнее запись.
Write-Behind: запись в кэш; асинхронный сброс в БД. Быстрая запись; риск потери данных.

### Redis: сценарии использования

- Хранение сессий
- Rate limiting (INCR + EXPIRE)
- Distributed lock (SET NX PX)
- Pub/Sub
- Лидерборды (Sorted Sets)
- Очереди задач (List LPUSH/BRPOP, Redis Streams)
- Кэш с TTL (cache-aside)

---
