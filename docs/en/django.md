# Django Internals

## QuerySet lazy evaluation

A QuerySet is not executed until it is evaluated. Evaluation happens when you: iterate, slice with step, call `list()`, `len()`, `bool()`, `repr()`, or pickle it.

```python
# No DB hit yet — just builds the query object
orders = Order.objects.filter(status='pending').order_by('-created_at')

# DB hit happens here (iteration)
for order in orders:
    print(order.id)

# DB hit: list()
all_orders = list(orders)

# DB hit: slicing without step (returns queryset, not list — still lazy!)
first_10 = orders[:10]            # still a QuerySet
list(orders[:10])                 # now hits DB

# _result_cache — after first evaluation, results are cached in the QuerySet
list(orders)    # DB hit
list(orders)    # no DB hit — uses _result_cache

# Force re-evaluation (clears cache)
orders._result_cache = None
```

**Common mistake:**

```python
# Two DB hits — QuerySet is evaluated twice
if Order.objects.filter(user=user).exists():       # hit 1
    orders = Order.objects.filter(user=user)        # hit 2 when iterated

# One hit — evaluate once, reuse
orders = list(Order.objects.filter(user=user))     # hit 1
if orders:                                          # no hit
    process(orders)
```

---

## Middleware

Middleware is a hook into Django's request/response processing. Each middleware wraps the view.

```
Request:  middleware_1 → middleware_2 → middleware_3 → view
Response: middleware_3 → middleware_2 → middleware_1 → client
```

```python
# Custom middleware — class-based
class TimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
        # one-time setup

    def __call__(self, request):
        import time
        start = time.perf_counter()

        response = self.get_response(request)   # call view (and inner middleware)

        duration = time.perf_counter() - start
        response['X-Response-Time'] = f'{duration:.3f}s'
        return response

    def process_exception(self, request, exception):
        # called if view raises an exception
        logger.error("Unhandled exception", exc_info=exception)
        return None   # None → Django continues normal exception handling

# settings.py
MIDDLEWARE = [
    'myapp.middleware.TimingMiddleware',
    'django.middleware.security.SecurityMiddleware',
    ...
]
```

---

## Signals

Decouple senders and receivers — sender doesn't need to know who's listening.

```python
from django.db.models.signals import post_save, pre_delete
from django.dispatch import receiver

@receiver(post_save, sender=Order)
def handle_order_created(sender, instance, created, **kwargs):
    if created:
        # transaction.on_commit to avoid task running before commit
        from django.db import transaction
        transaction.on_commit(
            lambda: send_confirmation.delay(instance.id)
        )

@receiver(pre_delete, sender=User)
def handle_user_deleted(sender, instance, **kwargs):
    cleanup_user_data(instance)

# Custom signals
from django.dispatch import Signal

order_shipped = Signal()   # define
order_shipped.send(sender=Order, order_id=42, tracking='ABC')   # send

@receiver(order_shipped)
def notify_customer(sender, order_id, tracking, **kwargs):
    ...
```

**Pitfalls:**
- Signals are synchronous — slow receivers slow the request
- Hard to trace execution flow
- Use `transaction.on_commit` inside `post_save` to avoid acting on uncommitted data
- Prefer explicit service layer calls for critical business logic

---

## Django ORM internals

```python
# Q objects — complex OR/AND/NOT queries
from django.db.models import Q

Order.objects.filter(
    Q(status='pending') | Q(status='processing'),
    user=request.user
)

# F expressions — reference model fields in queries (avoids race conditions)
from django.db.models import F

# Atomic increment — no Python read, no race condition
Order.objects.filter(pk=order_id).update(retry_count=F('retry_count') + 1)

# Subquery
from django.db.models import OuterRef, Subquery

latest_order = Order.objects.filter(
    user=OuterRef('pk')
).order_by('-created_at').values('status')[:1]

users = User.objects.annotate(latest_order_status=Subquery(latest_order))

# Prefetch with custom queryset
from django.db.models import Prefetch

orders = Order.objects.prefetch_related(
    Prefetch('items', queryset=OrderItem.objects.select_related('product'))
)
```

---

## FastAPI dependency injection

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session

app = FastAPI()

# Dependency — reusable, testable
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

# Route uses dependencies
@app.get("/orders/")
def list_orders(
    user: User = Depends(get_current_user),
    db: Session = Depends(get_db),
):
    return db.query(Order).filter(Order.user_id == user.id).all()

# In tests — override dependencies
app.dependency_overrides[get_db] = lambda: test_db_session
app.dependency_overrides[get_current_user] = lambda: fake_user
```

---

## Custom management commands

```python
# myapp/management/commands/sync_products.py
from django.core.management.base import BaseCommand, CommandError

class Command(BaseCommand):
    help = 'Sync products from external API'

    def add_arguments(self, parser):
        parser.add_argument('--dry-run', action='store_true', help='Preview only')
        parser.add_argument('--limit', type=int, default=100)

    def handle(self, *args, **options):
        dry_run = options['dry_run']
        limit   = options['limit']

        try:
            products = fetch_from_api(limit=limit)
        except APIError as e:
            raise CommandError(f'API failed: {e}')

        for product in products:
            if dry_run:
                self.stdout.write(f'Would sync: {product["name"]}')
            else:
                Product.objects.update_or_create(
                    external_id=product['id'],
                    defaults={'name': product['name'], 'price': product['price']}
                )

        self.stdout.write(self.style.SUCCESS(f'Synced {len(products)} products'))
```

```bash
python manage.py sync_products --dry-run
python manage.py sync_products --limit 50
```

---
