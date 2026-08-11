## 8. Architecture

### Microservices vs Monolith

**Monolith:**
+ Simple development, debugging, transactions, deploys
+ One codebase, easy refactoring
− Entire app deployed for any change
− Harder to scale individual parts
− Tech stack locked in

**Microservices:**
+ Independent scaling and deployment per service
+ Fault isolation, different tech stacks per service
+ Small focused teams own services end-to-end
− Network overhead, distributed transactions are complex
− Harder to test end-to-end
− Infrastructure complexity (service discovery, tracing, centralised logging)

**When to split:** when a monolith's deployment or scaling pain outweighs the coordination overhead. Don't start with microservices.

---

### Architectural patterns

**API Gateway:**
Single entry point for all clients. Handles auth, rate-limiting, routing, SSL termination, request transformation. Examples: Kong, AWS API Gateway, Nginx.

**Circuit Breaker:**
Stop calling a failing downstream service; return a fallback immediately. After a timeout, try again (half-open state). Prevents cascade failures.

```python
# Using the 'pybreaker' library
import pybreaker

breaker = pybreaker.CircuitBreaker(fail_max=5, reset_timeout=60)

@breaker
def call_payment_service(data):
    return requests.post('http://payments/charge', json=data, timeout=3)
```

**Saga:**
Distributed transaction via a sequence of local transactions with compensating actions on failure.

```
Order Service: create order → 
Payment Service: charge card → (fail) → 
Order Service: cancel order (compensating action)
```

Two approaches:
- Choreography: services emit events, others react (loose coupling, harder to trace)
- Orchestration: a saga orchestrator tells each service what to do (explicit flow)

**CQRS (Command Query Responsibility Segregation):**
Separate read and write models. Writes go to the command model (normalised, consistent); reads go to a query model (denormalised, optimised for display).

```python
# Command — changes state
class CreateOrderCommand:
    user_id: int
    items: list

# Query — reads state, never changes it
class GetOrdersByUserQuery:
    user_id: int
```

**Event Sourcing:**
Store state as an immutable sequence of events rather than current values.

```python
events = [
    OrderCreated(order_id=1, user_id=42),
    ItemAdded(order_id=1, product_id=7, qty=2),
    OrderShipped(order_id=1, tracking='ABC123'),
]
# Current state = replay of all events
```

Benefits: full audit log, easy time travel, event-driven integrations.
Tradeoffs: eventual consistency, query complexity, storage growth.

---

### Load balancing

**Algorithms:**
- Round-robin — requests distributed evenly in rotation
- Least connections — new request goes to server with fewest active connections
- IP hash — same client always hits same server (session affinity)
- Weighted — some servers get proportionally more traffic

**L4 vs L7:**
- L4 (transport layer) — TCP/UDP, fast, no content inspection (e.g. AWS NLB)
- L7 (application layer) — HTTP-aware, can route by path/header, SSL termination (e.g. Nginx, AWS ALB)

---

### Service mesh

Handles service-to-service communication: mTLS, retries, circuit breaking, tracing — without changing application code. Examples: Istio, Linkerd.

Each service gets a sidecar proxy (Envoy). The proxy intercepts all traffic.

---

### Observability

**The three pillars:**

```
Logs    → what happened                 (ELK, Loki, CloudWatch)
Metrics → how much / how fast           (Prometheus + Grafana)
Traces  → where time was spent          (Jaeger, Zipkin, OpenTelemetry)
```

**Structured logging:**

```python
import structlog
log = structlog.get_logger()
log.info("payment.processed", order_id=42, amount=99.99, duration_ms=145)
```

**SLI / SLO / SLA:**
- SLI (Service Level Indicator) — measured metric, e.g. p99 latency, error rate
- SLO (Service Level Objective) — target, e.g. 99.9% requests < 200ms
- SLA (Service Level Agreement) — contractual commitment with consequences

---

### 12-Factor App

1. Codebase — one repo, many deploys
2. Dependencies — explicitly declared (`requirements.txt`)
3. Config — in environment variables, never in code
4. Backing services — treat DB, Redis, S3 as attached resources (swappable via URL)
5. Build/release/run — strictly separated stages
6. Processes — stateless; store state in backing services
7. Port binding — app exports HTTP via a port
8. Concurrency — scale out via processes
9. Disposability — fast startup, graceful shutdown
10. Dev/prod parity — keep environments as similar as possible
11. Logs — treat as event streams (stdout → platform collects)
12. Admin processes — run as one-off processes (migrations, scripts)

---
