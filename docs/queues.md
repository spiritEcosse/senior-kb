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

---
