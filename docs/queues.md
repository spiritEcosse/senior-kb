## 7. Очереди сообщений и фоновые задачи

### Celery: надёжные задачи

**Базовая задача:**

```python
from celery import shared_task

@shared_task
def send_email(user_id: int):
    user = User.objects.get(pk=user_id)
    # ... отправка письма
```

**Запуск:**
```python
send_email.delay(user_id=42)
send_email.apply_async(args=[42], countdown=10)  # с задержкой
```

---

**Паттерн надёжности: transaction.on_commit**

Проблема: задача ставится в очередь до коммита транзакции → воркер получает задачу, но объекта в БД ещё нет.

```python
from django.db import transaction

def create_order(user_id, items):
    with transaction.atomic():
        order = Order.objects.create(user_id=user_id)
        OrderItem.objects.bulk_create([
            OrderItem(order=order, **item) for item in items
        ])
        # задача ставится ТОЛЬКО после успешного коммита
        transaction.on_commit(
            lambda: process_order.delay(order.id)
        )
```

---

**Идемпотентность + retry:**

```python
@shared_task(
    bind=True,           # self = экземпляр задачи
    max_retries=3,
    default_retry_delay=60,
    acks_late=True,      # подтверждение после успешного выполнения
)
def process_payment(self, payment_id: int):
    try:
        payment = Payment.objects.get(pk=payment_id)
        if payment.status == 'completed':
            return  # идемпотентность: уже обработано
        charge_card(payment)
        payment.status = 'completed'
        payment.save()
    except TemporaryError as exc:
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)
```

`bind=True` → доступ к `self.retry()`, `self.request.retries`.
`acks_late=True` → сообщение подтверждается брокеру только после выполнения (защита от потери при падении воркера).

---

### RabbitMQ vs Kafka

RabbitMQ: брокер сообщений; push-модель; сообщения удаляются после потребления; хорош для task queues и маршрутизации.
Kafka: распределённый лог; pull-модель; сообщения хранятся настраиваемое время; хорош для event streaming, audit log, высокопропускных пайплайнов; консьюмеры сами отслеживают offset.

---
