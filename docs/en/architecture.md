## 8. Architecture

### Microservices vs Monolith

Microservices:
+ Independent scaling and deployment
+ Fault isolation, different tech stacks
- Network overhead, complex distributed transactions
- Complex infrastructure (service discovery, tracing, centralised logging)

Monolith:
+ Simple development, debugging, and transactions
- Entire app deployed for any change
- Harder to scale individual parts

### Architectural Patterns

- API Gateway: single entry point; auth, rate-limiting, routing.
- Circuit Breaker: stop calling a failing service; fallback; recover after timeout.
- Saga: distributed transaction via compensating operations.
- CQRS: separate read and write models.
- Event Sourcing: store state as a sequence of events.

---
