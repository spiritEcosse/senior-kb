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
