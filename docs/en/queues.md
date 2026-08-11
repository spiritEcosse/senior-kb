## 7. Message Queues and Background Tasks

### Celery

Distributed task queue for Python. Workers pick tasks from a broker (Redis, RabbitMQ).
`@shared_task` — declare a task. `task.delay()` / `task.apply_async()` — enqueue.
Use cases: sending emails, heavy computation outside the request cycle, periodic tasks (beat).

### RabbitMQ vs Kafka

RabbitMQ: message broker; push model; messages deleted after consumption; great for task queues and routing.
Kafka: distributed log; pull model; messages retained for a configurable period; great for event streaming, audit logs, high-throughput pipelines; consumers track their own offset.

---
