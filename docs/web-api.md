## 5. Web and API

### REST vs HTTP

HTTP — protocol. REST — architectural style on top of HTTP.
REST: stateless, CRUD via HTTP verbs, resources via URIs.

### Stateless vs Stateful

Stateless: each request is self-contained; JWT in header; easy to scale horizontally.
Stateful: server stores session; harder to scale.

### GraphQL

Client requests exactly the fields it needs. Single endpoint (usually `POST /graphql`). Strict schema typing.
Pros: no over/under-fetching, introspection (GraphiQL/GraphQL Playground), one round trip for nested/related data.
Cons: harder HTTP caching (single endpoint, `POST`-based), N+1 (→ DataLoader), harder monitoring/rate-limiting per-field cost, file uploads need extensions.

**Schema — types, queries, mutations, subscriptions:**

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  orders: [Order!]!          # nested/related field — resolved separately
}

type Order {
  id: ID!
  total: Float!
  status: OrderStatus!
}

enum OrderStatus { PENDING SHIPPED DELIVERED }

type Query {
  user(id: ID!): User
  users(first: Int, after: String): UserConnection!
}

type Mutation {
  createOrder(userId: ID!, productId: ID!, qty: Int!): Order!
}

type Subscription {
  orderStatusChanged(orderId: ID!): Order!    # WebSocket, real-time push
}
```

```graphql
# Client sends exactly what it needs — no more, no less
query {
  user(id: "42") {
    name
    orders { id total status }
  }
}

mutation {
  createOrder(userId: "42", productId: "7", qty: 2) {
    id
    status
  }
}
```

**Resolvers** — one function per field; GraphQL walks the query tree and calls a resolver for each requested field:

```python
# Python, Strawberry
import strawberry

@strawberry.type
class Order:
    id: strawberry.ID
    total: float

@strawberry.type
class User:
    id: strawberry.ID
    name: str

    @strawberry.field
    def orders(self) -> list[Order]:
        return order_repo.get_by_user(self.id)   # runs once PER user in the result set

@strawberry.type
class Query:
    @strawberry.field
    def user(self, id: strawberry.ID) -> User | None:
        return user_repo.get(id)
```

**N+1 problem and DataLoader:**

Requesting `users { orders { ... } }` for 100 users naively fires 1 query for users + 100 queries for orders (one per resolver call). `DataLoader` batches and caches those per-request:

```python
from strawberry.dataloader import DataLoader

async def batch_load_orders(user_ids: list[int]) -> list[list[Order]]:
    # ONE query for all requested users, then group in memory
    orders = await Order.objects.filter(user_id__in=user_ids)
    by_user = defaultdict(list)
    for o in orders:
        by_user[o.user_id].append(o)
    return [by_user[uid] for uid in user_ids]   # must match input order

order_loader = DataLoader(load_fn=batch_load_orders)

@strawberry.type
class User:
    id: strawberry.ID

    @strawberry.field
    async def orders(self) -> list[Order]:
        return await order_loader.load(self.id)   # queued, batched, deduped
```

DataLoader collects all `.load()` calls within one tick of the event loop, then issues a single batched call — turns N+1 into 2 queries. Scope one `DataLoader` instance per request (never share/cache across requests — stale/leaked data).

**REST vs GraphQL:**

| | REST | GraphQL |
|---|---|---|
| Endpoints | Many (one per resource) | One |
| Fetching | Fixed shape — over/under-fetching | Client picks exact fields |
| Versioning | `/v1/`, `/v2/` | Evolve schema — deprecate fields instead |
| HTTP caching | Native (GET + URL as cache key) | Needs app-level caching (Apollo cache, persisted queries) |
| Status codes | 200/400/404/500 | Usually 200, errors in body `errors[]` |
| File upload | Native (multipart) | Needs `graphql-multipart-request-spec` extension |
| Learning curve | Low | Higher (schema, resolvers, N+1) |

**Pagination — Relay cursor connections** (the GraphQL-standard shape):

```graphql
type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
}

type UserEdge {
  cursor: String!
  node: User!
}

type PageInfo {
  hasNextPage: Boolean!
  endCursor: String
}
```

```graphql
query {
  users(first: 20, after: "cursor123") {
    edges { cursor node { id name } }
    pageInfo { hasNextPage endCursor }
  }
}
```

**Security — a GraphQL query is arbitrary client-defined workload, so guard it explicitly:**

- **Query depth limiting** — reject deeply nested queries (`user { orders { user { orders { ... } } } }`) that could recurse expensively.
- **Query complexity/cost analysis** — assign a cost per field, reject queries above a budget (protects against a single request fanning out into thousands of resolver calls).
- **Disable introspection in production** — schema introspection (`__schema`, `__type`) is invaluable in dev (powers GraphiQL) but hands attackers your full API surface.
- **Persisted queries** — client sends a hash instead of the full query string; server only executes pre-registered queries. Cuts payload size and closes off arbitrary ad-hoc queries.
- **Per-field authorization** — REST checks auth at the endpoint; GraphQL needs it per-resolver/per-field, since one query can touch many types with different access rules.

**Python ecosystem:** Strawberry and Graphene (standalone or `strawberry-django` / `graphene-django` for Django model integration); FastAPI commonly pairs with Strawberry.

### JWT (JSON Web Token)

Structure: `header.payload.signature` — each part is Base64Url-encoded, separated by dots.

**Header** — algorithm and token type:
```json
{"alg": "HS256", "typ": "JWT"}
```

**Payload** — claims (never put secrets here — encoded, not encrypted):

| Claim | Name | Description |
|-------|------|-------------|
| `sub` | Subject | Who the token is about — usually user ID |
| `iat` | Issued At | Unix timestamp when token was created |
| `exp` | Expiration | Unix timestamp after which token is invalid |
| `nbf` | Not Before | Token invalid before this timestamp |
| `iss` | Issuer | Who issued the token |
| `aud` | Audience | Intended recipient(s) |
| `jti` | JWT ID | Unique ID — prevents replay attacks |

**Signature** — `ALGORITHM(base64url(header) + "." + base64url(payload), key)`:

- **HS256** — symmetric: one shared secret signs and verifies. Fast, but every verifier can also forge tokens.
- **RS256** — asymmetric: private key signs, public key verifies. Verifiers cannot forge tokens. Use with JWKS endpoint.

| | HS256 | RS256 |
|---|---|---|
| Key | 1 shared secret | Private + Public pair |
| Verifier can forge? | Yes | No |
| JWKS endpoint | ✗ | ✓ |
| Best for | Monolith / single service | Microservices / third parties |

Stateless: server only needs the secret/public key to verify — no DB lookup.
Risk: token cannot be revoked before expiry without a blocklist (e.g. Redis).

```python
import jwt
from jwt import PyJWKClient

# HS256 — sign and verify
token = jwt.encode(
    {"sub": "user_42", "exp": 1723500000, "iat": 1723496400},
    "secret", algorithm="HS256"
)
payload = jwt.decode(token, "secret", algorithms=["HS256"])

# RS256 — verify via JWKS
jwks_client = PyJWKClient("https://auth.myapp.com/.well-known/jwks.json")
signing_key = jwks_client.get_signing_key_from_jwt(token)
payload = jwt.decode(token, signing_key.key, algorithms=["RS256"], audience="api.myapp.com")

# Decode payload without verification (inspect only — never trust unverified)
import base64, json
part = token.split(".")[1] + "=="
print(json.loads(base64.urlsafe_b64decode(part)))
```

**Token refresh pattern:**

- `access_token` — short-lived (15 min), stateless
- `refresh_token` — long-lived (7–30 days), stored server-side (DB/Redis), rotated on use
- On expiry: client sends refresh_token → server validates → issues new access_token

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

---

### Deprecation warnings via headers

When deprecating an API endpoint, signal it to clients via response headers — many API gateways, SDKs, and monitoring tools read these automatically:

```python
# FastAPI example
from fastapi import Response
from fastapi.responses import JSONResponse

@app.get("/api/v1/users/{id}")
async def get_user_v1(id: int, response: Response):
    response.headers["Deprecation"] = "true"
    response.headers["Sunset"] = "Sat, 31 Dec 2025 23:59:59 GMT"   # RFC 7231 date
    response.headers["Link"] = '</api/v2/users/{id}>; rel="successor-version"'
    response.headers["Warning"] = '299 - "This endpoint is deprecated. Use /api/v2/users/"'
    return get_user_data(id)
```

**Standard headers:**

| Header | Purpose | Who reads it |
|---|---|---|
| `Deprecation: true` | Marks endpoint as deprecated (RFC 8594) | API clients, SDKs |
| `Sunset: <date>` | When the endpoint will be removed | Monitoring, client devs |
| `Link: <url>; rel="successor-version"` | Points to replacement | Automated tools |
| `Warning: 299 - "message"` | Human-readable warning | Browsers, curl, logging |

**Yes — other servers and tools do read these:**
- API gateways (Kong, AWS API Gateway) can alert on `Deprecation` headers
- OpenAPI generators surface them in docs
- SDKs like the GitHub client emit log warnings when they see `Deprecation`
- Monitoring systems (Datadog, Grafana) can track deprecation header rates

---

### gRPC

**What it is:** Remote Procedure Call framework by Google. Uses HTTP/2 as transport and Protocol Buffers (protobuf) as the serialization format.

**Why use it over REST:**

| | REST/JSON | gRPC |
|---|---|---|
| Serialization | JSON (text, verbose) | Protobuf (binary, ~5–10× smaller) |
| Speed | Slower | Faster (binary + HTTP/2 multiplexing) |
| Schema | Optional (OpenAPI) | Mandatory `.proto` — strong contract |
| Streaming | Workarounds (SSE, WebSocket) | Native bi-directional streaming |
| Browser support | Native | Needs grpc-web proxy |
| Use case | Public APIs, browser clients | Internal microservices |

**Define a service in `.proto`:**

```protobuf
// user_service.proto
syntax = "proto3";

service UserService {
  rpc GetUser      (GetUserRequest)    returns (User);
  rpc ListUsers    (ListUsersRequest)  returns (stream User);     // server streaming
  rpc CreateUser   (CreateUserRequest) returns (User);
}

message GetUserRequest { int32 id = 1; }
message ListUsersRequest { int32 page = 1; int32 page_size = 2; }
message CreateUserRequest { string name = 1; string email = 2; }
message User { int32 id = 1; string name = 2; string email = 3; }
```

**Python server (grpcio):**

```python
import grpc
from concurrent import futures
import user_service_pb2, user_service_pb2_grpc

class UserServicer(user_service_pb2_grpc.UserServiceServicer):
    def GetUser(self, request, context):
        user = db.get_user(request.id)
        if not user:
            context.set_code(grpc.StatusCode.NOT_FOUND)
            context.set_details("User not found")
            return user_service_pb2.User()
        return user_service_pb2.User(id=user.id, name=user.name, email=user.email)

    def ListUsers(self, request, context):
        for user in db.list_users(page=request.page, page_size=request.page_size):
            yield user_service_pb2.User(id=user.id, name=user.name)

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    user_service_pb2_grpc.add_UserServiceServicer_to_server(UserServicer(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()
```

**Python client:**

```python
import grpc
import user_service_pb2, user_service_pb2_grpc

with grpc.insecure_channel('localhost:50051') as channel:
    stub = user_service_pb2_grpc.UserServiceStub(channel)
    user = stub.GetUser(user_service_pb2.GetUserRequest(id=42))
    print(user.name)

    # Server streaming
    for user in stub.ListUsers(user_service_pb2.ListUsersRequest(page=1, page_size=10)):
        print(user.name)
```

**gRPC streaming types:**

| Type | Direction | Use case |
|---|---|---|
| Unary | request → response | Standard call |
| Server streaming | request → stream of responses | Real-time feeds, large result sets |
| Client streaming | stream of requests → response | File upload, batch insert |
| Bidirectional | stream ↔ stream | Chat, live collaboration |

**Code generation:**

```bash
pip install grpcio grpcio-tools
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. user_service.proto
```

---

### FastAPI — Business logic in repositories, not endpoints

**Why:** Single Responsibility Principle. Each layer has one job:

- **Endpoint** — HTTP: parse request, validate input (Pydantic), call service/repo, return response
- **Repository** — data access only: SQL/Mongo queries, no business rules
- **Service** (optional) — orchestrates repositories, contains business logic

**Bad — logic in endpoint:**

```python
@app.post("/orders/")
async def create_order(data: OrderCreate, db: AsyncSession = Depends(get_db)):
    # business logic + DB access mixed in endpoint
    if data.quantity <= 0:
        raise HTTPException(400, "Quantity must be positive")
    product = await db.get(Product, data.product_id)
    if not product or product.stock < data.quantity:
        raise HTTPException(400, "Insufficient stock")
    product.stock -= data.quantity
    order = Order(user_id=data.user_id, product_id=data.product_id, qty=data.quantity)
    db.add(order)
    await db.commit()
    return order
```

**Good — repository pattern:**

```python
# repositories/order_repo.py
class OrderRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def create(self, user_id: int, product_id: int, qty: int) -> Order:
        order = Order(user_id=user_id, product_id=product_id, qty=qty)
        self.session.add(order)
        await self.session.flush()   # get ID without committing
        return order

    async def get_by_user(self, user_id: int) -> list[Order]:
        result = await self.session.execute(
            select(Order).where(Order.user_id == user_id)
        )
        return result.scalars().all()

# services/order_service.py
class OrderService:
    def __init__(self, order_repo: OrderRepository, product_repo: ProductRepository):
        self.orders = order_repo
        self.products = product_repo

    async def create_order(self, user_id: int, product_id: int, qty: int) -> Order:
        product = await self.products.get(product_id)
        if not product or product.stock < qty:
            raise InsufficientStockError(product_id)
        await self.products.decrement_stock(product_id, qty)
        return await self.orders.create(user_id, product_id, qty)

# endpoints/orders.py
@router.post("/orders/", response_model=OrderResponse)
async def create_order(
    data: OrderCreate,
    service: OrderService = Depends(get_order_service)
):
    try:
        order = await service.create_order(data.user_id, data.product_id, data.quantity)
        return order
    except InsufficientStockError as e:
        raise HTTPException(status_code=400, detail=str(e))
```

**Benefits:**
- Endpoints stay thin — easy to read, easy to version
- Business logic is testable without HTTP layer
- Repository is swappable (swap Postgres for MongoDB in tests)
- Service can be reused by CLI commands, background tasks, not just HTTP

---


