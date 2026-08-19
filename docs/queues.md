## 7. Message Queues and Background Tasks

### Celery: Reliable Tasks

**Basic task:**

```python
from celery import shared_task

@shared_task
def send_email(user_id: int):
    user = User.objects.get(pk=user_id)
    # ... send email
```

**Enqueuing:**
```python
send_email.delay(user_id=42)
send_email.apply_async(args=[42], countdown=10)  # with delay
```

---

**Reliability pattern: transaction.on_commit**

Problem: task is enqueued before the transaction commits → worker gets the task but the DB object doesn't exist yet.

```python
from django.db import transaction

def create_order(user_id, items):
    with transaction.atomic():
        order = Order.objects.create(user_id=user_id)
        OrderItem.objects.bulk_create([
            OrderItem(order=order, **item) for item in items
        ])
        # task enqueued ONLY after successful commit
        transaction.on_commit(
            lambda: process_order.delay(order.id)
        )
```

---

**Idempotency + retry:**

```python
@shared_task(
    bind=True,           # self = task instance
    max_retries=3,
    default_retry_delay=60,
    acks_late=True,      # acknowledge only after success
)
def process_payment(self, payment_id: int):
    try:
        payment = Payment.objects.get(pk=payment_id)
        if payment.status == 'completed':
            return  # idempotency: already processed
        charge_card(payment)
        payment.status = 'completed'
        payment.save()
    except TemporaryError as exc:
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)
```

`bind=True` → access to `self.retry()`, `self.request.retries`.
`acks_late=True` → message acknowledged to broker only after execution (protects against loss on worker crash).

---

### RabbitMQ vs Kafka

RabbitMQ: message broker; push model; messages deleted after consumption; great for task queues and routing.
Kafka: distributed log; pull model; messages retained for a configurable period; great for event streaming, audit logs, high-throughput pipelines; consumers track their own offset.

**What "high-throughput pipeline" means in practice:**

A pipeline is a chain of processing steps where data flows from producers through Kafka topics to one or more consumers that transform, aggregate, or store it. Each consumer reads independently at its own pace.

```
# Event streaming — one topic, multiple independent consumers
User action (click, purchase)
  → Kafka topic: user-events
      → Consumer 1: update real-time analytics dashboard
      → Consumer 2: trigger recommendation engine
      → Consumer 3: write to data warehouse (Redshift, BigQuery)

# ETL / data lake ingestion
PostgreSQL row changes (Debezium CDC)
  → Kafka topic: db-changes
      → Flink/Spark Streaming
          → S3 / Parquet (data lake)
# No bulk exports needed — every DB write streams automatically

# Log aggregation
100 app servers
  → Kafka topic: app-logs    (buffers the firehose)
      → Logstash / Flink
          → Elasticsearch
# If ES goes down: logs accumulate in Kafka, replay when it recovers

# High-frequency market data — high throughput, built for exactly this
Exchange price ticks (thousands/sec per symbol)
  → Kafka topic: market-data.{symbol}
      → Consumer 1: order book / pricing engine
      → Consumer 2: real-time charting service
      → Consumer 3: archive to cold storage for backtesting
# Sequential disk writes + partitioning per symbol → millions of msg/sec,
# no consumer competes with another for the same tick

# Fraud detection — sliding window over a stream
Payment events
  → Kafka
      → Flink (rule: >3 transactions in 60s from same card)
          → alert topic
              → fraud service

# Microservice event bus
Order Service publishes: OrderCreated
  → Kafka
      → Inventory Service:    reserve stock
      → Notification Service: send confirmation email
      → Analytics Service:    update dashboard
```

**Why Kafka fits these and RabbitMQ doesn't scale as well:**

| | RabbitMQ | Kafka |
|---|---|---|
| Message retention | Deleted after consumption | Days/weeks — full replay possible |
| Multiple consumers | Competing (one gets the message) | Independent — each reads at own offset |
| Throughput | ~50k msg/sec | Millions msg/sec (sequential disk writes) |
| Backpressure | Slow consumer blocks the queue | Slow consumer just falls behind on offset |
| Use case | Task queues, RPC, routing | Event streaming, audit log, data pipelines |

---

**Kafka Consumer Groups vs Redis Pub/Sub**

Redis pub/sub has no concept of offset — a subscriber only gets messages published while it's connected. Kafka fixes that with consumer groups:

- Each service subscribes as its own consumer group (e.g. `analytics-service`, `notification-service`)
- Kafka tracks the offset (read position) per consumer group independently, stored in Kafka itself
- Every consumer group gets its own full copy of every message on the topic, each at its own pace

```
Order Service publishes: OrderCreated → Kafka topic: order-events

  Consumer group "analytics-service"     → offset 1042
  Consumer group "notification-service"  → offset 1042
```

If Analytics Service goes down for 2 minutes, it resumes from its last committed offset when it comes back — no messages lost, no need to replay from Notification Service's position.

What this buys you over Redis pub/sub:

1. **Persistence** — messages are stored on disk, not lost if a consumer is offline
2. **Consumer groups** — multiple independent consumers each get every message, at their own offset
3. **Replay** — rewind a consumer group's offset to reprocess historical messages (critical for fintech audits)

---
