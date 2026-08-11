## 5. Web and API

### REST vs HTTP

HTTP — protocol. REST — architectural style on top of HTTP.
REST: stateless, CRUD via HTTP verbs, resources via URIs.

### Stateless vs Stateful

Stateless: each request is self-contained; JWT in header; easy to scale horizontally.
Stateful: server stores session; harder to scale.

### GraphQL

Client requests exactly the fields it needs. Single endpoint. Strict schema typing.
Pros: no over/under-fetching, introspection (GraphiQL).
Cons: harder HTTP caching, N+1 (→ DataLoader), harder monitoring.

### JWT (JSON Web Token)

Structure: `header.payload.signature` (base64url).
Header — algorithm (HS256/RS256). Payload — claims (sub, exp, iat). Signature — HMAC or RSA.
Stateless: server only needs the secret/public key to verify.
Risk: token cannot be revoked before expiry without a blocklist.

### OAuth 2.0

Authorization Code (web apps): redirect → code → exchange for access_token + refresh_token.
Client Credentials (M2M): client_id + client_secret → access_token.
PKCE: protects Authorization Code for public clients (SPA, mobile).
OpenID Connect adds an identity layer on top of OAuth (id_token).

### SSO (Single Sign-On)

One login — access to many systems without re-authenticating.
Typically: OAuth 2.0 + OpenID Connect.

### CSRF in Django

1. Generates a token for the session
2. Token in the form (hidden field) + `csrftoken` cookie
3. On submit: cookie vs `X-CSRFToken` header compared
4. Mismatch → 403

🔗 [https://telegra.ph/Django-CSRF-Rukovodstvo-08-04](https://telegra.ph/Django-CSRF-Rukovodstvo-08-04)

### WSGI vs ASGI

WSGI: synchronous; `callable(environ, start_response)`. Servers: Gunicorn, uWSGI.
ASGI: asynchronous; supports WebSocket, long-polling. Servers: Uvicorn, Daphne.

### API Security: IDOR

```python
# Vulnerable: user can supply another user's order_id
@api_view(['GET'])
def get_order(request, order_id):
    order = Order.objects.get(pk=order_id)  # IDOR!
    return Response(OrderSerializer(order).data)

# Correct: filter by current user
@api_view(['GET'])
@permission_classes([IsAuthenticated])
def get_order(request, order_id):
    order = get_object_or_404(Order, pk=order_id, user=request.user)
    return Response(OrderSerializer(order).data)
```

### .exists() vs bool(queryset)

```python
# Slow: loads all objects into memory
if Order.objects.filter(user=user):
    ...

# Fast: SELECT 1 LIMIT 1
if Order.objects.filter(user=user).exists():
    ...
```

### DNS + Full HTTP Request Lifecycle

1. Check local DNS cache
2. Recursive DNS: local resolver → root NS → TLD NS → authoritative NS → IP
3. Cached by TTL
4. TCP 3-way handshake (SYN, SYN-ACK, ACK) → port 80/443
5. TLS handshake for HTTPS
6. HTTP request → server response

### TCP vs UDP

TCP: reliable, acknowledged, ordered. HTTP, SSH, FTP.
UDP: fast, no guarantees. DNS, video, games, VoIP.

### OSI Model (7 layers)

7. Application (HTTP, DNS, FTP)
6. Presentation (SSL/TLS)
5. Session
4. Transport (TCP, UDP)
3. Network (IP)
2. Data Link (Ethernet)
1. Physical

---

---

## API Design

### Versioning strategies

```python
# 1. URI versioning — most common, easy to test in browser
GET /api/v1/orders/
GET /api/v2/orders/

# 2. Header versioning — cleaner URLs, harder to test
GET /api/orders/
Accept: application/vnd.myapp.v2+json

# 3. Query param — simplest, least RESTful
GET /api/orders/?version=2
```

| Strategy | Pros | Cons |
|---|---|---|
| URI | Explicit, cacheable, easy to test | URL proliferation |
| Header | Clean URLs | Harder to discover, test |
| Query param | Simple | Not RESTful, pollutes URLs |

---

### Pagination

**Offset pagination** — simple but has issues at scale:

```python
# GET /api/orders/?offset=100&limit=20
queryset = Order.objects.all()[100:120]

# Problem: if a new order is inserted at offset 50 while paginating,
# you skip one item or see a duplicate
```

**Cursor pagination** — stable, performant, no duplicates:

```python
# GET /api/orders/?cursor=eyJpZCI6IDEwMH0&limit=20
# cursor encodes: {"id": 100, "created_at": "2024-01-15T10:00:00"}

import base64, json

def encode_cursor(order):
    payload = {"id": order.id, "created_at": order.created_at.isoformat()}
    return base64.urlsafe_b64encode(json.dumps(payload).encode()).decode()

def decode_cursor(cursor):
    return json.loads(base64.urlsafe_b64decode(cursor.encode()))

# Query
def get_page(cursor=None, limit=20):
    qs = Order.objects.order_by('-created_at', '-id')
    if cursor:
        data = decode_cursor(cursor)
        qs = qs.filter(
            created_at__lt=data['created_at']
        ) | qs.filter(
            created_at=data['created_at'], id__lt=data['id']
        )
    orders = list(qs[:limit + 1])   # fetch one extra to detect next page
    has_next = len(orders) > limit
    return orders[:limit], encode_cursor(orders[limit - 1]) if has_next else None
```

---

### Idempotency keys

For non-idempotent operations (payments, order creation) — allow safe retries:

```python
# Client sends: POST /api/orders/  Idempotency-Key: uuid4-from-client
# Server stores result keyed by (user_id, idempotency_key)
# If same key arrives again → return cached result, don't re-execute

from django.core.cache import cache

def create_order(request):
    key = request.headers.get('Idempotency-Key')
    if key:
        cached = cache.get(f'idem:{request.user.id}:{key}')
        if cached:
            return JsonResponse(cached, status=200)

    order = OrderService.create_order(request.user, request.data['items'])
    response = {'id': order.id, 'status': order.status}

    if key:
        cache.set(f'idem:{request.user.id}:{key}', response, timeout=86400)

    return JsonResponse(response, status=201)
```

---

### Webhooks

Push model: your server calls the client's endpoint when an event occurs.

```python
# Signing webhook payloads — client verifies authenticity
import hmac, hashlib, time

def send_webhook(url: str, payload: dict, secret: str):
    body = json.dumps(payload).encode()
    timestamp = str(int(time.time()))
    signature = hmac.new(
        secret.encode(), f"{timestamp}.".encode() + body, hashlib.sha256
    ).hexdigest()

    requests.post(url, data=body, headers={
        'Content-Type': 'application/json',
        'X-Timestamp': timestamp,
        'X-Signature': f'sha256={signature}',
    }, timeout=10)

# Client verifies:
def verify_webhook(body: bytes, timestamp: str, signature: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode(), f"{timestamp}.".encode() + body, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f'sha256={expected}', signature)
```

**Webhook best practices:**
- Always sign payloads (HMAC-SHA256)
- Verify timestamp to prevent replay attacks (reject if > 5 min old)
- Respond 200 immediately; process async (via Celery)
- Implement retry with exponential backoff on the sender side
- Make handlers idempotent (same event delivered twice = same result)

---
