# Django: внутреннее устройство

## Ленивые QuerySet-ы

QuerySet не выполняется до момента обращения. Обращение происходит при: итерации, срезе с шагом, вызове `list()`, `len()`, `bool()`, `repr()`, пикл.

```python
# Нет запроса в БД — только строится объект запроса
orders = Order.objects.filter(status='pending').order_by('-created_at')

# Запрос выполняется здесь (итерация)
for order in orders:
    print(order.id)

# Запрос: list()
all_orders = list(orders)

# Срез без шага — всё ещё QuerySet (ленивый!)
first_10 = orders[:10]     # QuerySet
list(orders[:10])           # теперь запрос

# _result_cache — после первого обращения результаты кэшируются
list(orders)    # запрос
list(orders)    # нет запроса — используется _result_cache
```

**Типичная ошибка:**

```python
# Два запроса — QuerySet вычисляется дважды
if Order.objects.filter(user=user).exists():       # запрос 1
    orders = Order.objects.filter(user=user)        # запрос 2 при итерации

# Один запрос — вычислить один раз, переиспользовать
orders = list(Order.objects.filter(user=user))     # запрос 1
if orders:                                          # нет запроса
    process(orders)
```

---

## Middleware

Middleware — хук в обработку запроса/ответа Django. Каждый middleware оборачивает view.

```
Запрос:  middleware_1 → middleware_2 → middleware_3 → view
Ответ:   middleware_3 → middleware_2 → middleware_1 → клиент
```

```python
class TimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        import time
        start = time.perf_counter()

        response = self.get_response(request)   # вызов view и внутренних middleware

        duration = time.perf_counter() - start
        response['X-Response-Time'] = f'{duration:.3f}s'
        return response

    def process_exception(self, request, exception):
        logger.error("Необработанное исключение", exc_info=exception)
        return None   # None → Django продолжает стандартную обработку
```

---

## Signals

Развязывают отправителей и получателей — отправитель не знает кто слушает.

```python
from django.db.models.signals import post_save, pre_delete
from django.dispatch import receiver

@receiver(post_save, sender=Order)
def handle_order_created(sender, instance, created, **kwargs):
    if created:
        from django.db import transaction
        transaction.on_commit(
            lambda: send_confirmation.delay(instance.id)
        )

# Кастомные сигналы
from django.dispatch import Signal

order_shipped = Signal()
order_shipped.send(sender=Order, order_id=42, tracking='ABC')

@receiver(order_shipped)
def notify_customer(sender, order_id, tracking, **kwargs):
    ...
```

**Подводные камни:**
- Signals синхронные — медленный receiver замедляет запрос
- Трудно отслеживать поток выполнения
- Используй `transaction.on_commit` внутри `post_save` — иначе задача может запуститься до коммита
- Предпочитай явный вызов сервисного слоя для критической бизнес-логики

---

## Django ORM: продвинутые запросы

```python
# Q-объекты — сложные OR/AND/NOT запросы
from django.db.models import Q

Order.objects.filter(
    Q(status='pending') | Q(status='processing'),
    user=request.user
)

# F-выражения — ссылка на поля модели в запросах (атомарные операции)
from django.db.models import F

# Атомарный инкремент — нет Python-чтения, нет race condition
Order.objects.filter(pk=order_id).update(retry_count=F('retry_count') + 1)

# Подзапрос
from django.db.models import OuterRef, Subquery

latest_order = Order.objects.filter(
    user=OuterRef('pk')
).order_by('-created_at').values('status')[:1]

users = User.objects.annotate(latest_order_status=Subquery(latest_order))

# Prefetch с кастомным queryset
from django.db.models import Prefetch

orders = Order.objects.prefetch_related(
    Prefetch('items', queryset=OrderItem.objects.select_related('product'))
)
```

---

## FastAPI: dependency injection

```python
from fastapi import FastAPI, Depends, HTTPException

app = FastAPI()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db)
) -> User:
    user = verify_token(token, db)
    if not user:
        raise HTTPException(status_code=401)
    return user

@app.get("/orders/")
def list_orders(
    user: User = Depends(get_current_user),
    db: Session = Depends(get_db),
):
    return db.query(Order).filter(Order.user_id == user.id).all()

# В тестах — переопределить зависимости
app.dependency_overrides[get_db] = lambda: test_db_session
app.dependency_overrides[get_current_user] = lambda: fake_user
```

---

## Кастомные management команды

```python
# myapp/management/commands/sync_products.py
from django.core.management.base import BaseCommand, CommandError

class Command(BaseCommand):
    help = 'Синхронизация продуктов из внешнего API'

    def add_arguments(self, parser):
        parser.add_argument('--dry-run', action='store_true')
        parser.add_argument('--limit', type=int, default=100)

    def handle(self, *args, **options):
        try:
            products = fetch_from_api(limit=options['limit'])
        except APIError as e:
            raise CommandError(f'API вернул ошибку: {e}')

        for product in products:
            if options['dry_run']:
                self.stdout.write(f'Синхронизирую: {product["name"]}')
            else:
                Product.objects.update_or_create(
                    external_id=product['id'],
                    defaults={'name': product['name'], 'price': product['price']}
                )

        self.stdout.write(self.style.SUCCESS(f'Синхронизировано {len(products)} продуктов'))
```

---
