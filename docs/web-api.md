## 5. Web и API

### REST vs HTTP

HTTP — протокол. REST — архитектурный стиль поверх HTTP.
REST: stateless, CRUD через HTTP-глаголы, ресурсы через URI.

### Stateless vs Stateful

Stateless: каждый запрос самодостаточен; JWT в заголовке; легко масштабировать горизонтально.
Stateful: сервер хранит сессию; труднее масштабировать.

### GraphQL

Клиент запрашивает ровно нужные поля. Один endpoint. Строгая типизация схемы.
Плюсы: нет over/under-fetching, интроспекция (GraphiQL).
Минусы: сложнее HTTP-кэширование, N+1 (→ DataLoader), сложнее мониторинг.

### JWT (JSON Web Token)

Структура: header.payload.signature (base64url).
Header — алгоритм (HS256/RS256). Payload — claims (sub, exp, iat). Signature — HMAC или RSA.
Stateless: серверу нужен только секрет/публичный ключ для проверки.
Риск: токен нельзя отозвать до истечения срока без blocklist.

### OAuth 2.0

Authorization Code (веб-приложения): редирект → code → обмен на access_token + refresh_token.
Client Credentials (M2M): client_id + client_secret → access_token.
PKCE: защита Authorization Code для публичных клиентов (SPA, мобильные).
OpenID Connect добавляет слой идентификации поверх OAuth (id_token).

### SSO (Single Sign-On)

Один вход — доступ ко многим системам без повторной аутентификации.
Обычно: OAuth 2.0 + OpenID Connect.

### CSRF в Django

1. Генерирует токен для сессии
2. Токен в форме (скрытое поле) + cookie csrftoken
3. При отправке: сравнение cookie и X-CSRFToken
4. Несовпадение → 403

🔗 [https://telegra.ph/Django-CSRF-Rukovodstvo-08-04](https://telegra.ph/Django-CSRF-Rukovodstvo-08-04)

### WSGI vs ASGI

WSGI: синхронный; callable(environ, start_response). Серверы: Gunicorn, uWSGI.
ASGI: асинхронный; поддерживает WebSocket, long-polling. Серверы: Uvicorn, Daphne.

### DNS + полный цикл HTTP-запроса

1. Проверить локальный DNS-кэш
2. Рекурсивный DNS: локальный resolver → root NS → TLD NS → авторитетный NS → IP
3. Кэш по TTL
4. TCP 3-way handshake (SYN, SYN-ACK, ACK) → порт 80/443
5. TLS-рукопожатие для HTTPS
6. HTTP-запрос → ответ сервера

### TCP vs UDP

TCP: надёжно, с подтверждением, с порядком. HTTP, SSH, FTP.
UDP: быстро, без гарантий. DNS, видео, игры, VoIP.

### OSI-модель (7 уровней)

7. Прикладной (HTTP, DNS, FTP)
6. Представления (SSL/TLS)
5. Сеансовый
4. Транспортный (TCP, UDP)
3. Сетевой (IP)
2. Канальный (Ethernet)
1. Физический

---
