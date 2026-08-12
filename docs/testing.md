## 9. Testing and Code Quality

### Types of testing

Unit: isolated unit; no external dependencies; maximum speed.
Integration: component interaction with real dependencies.
Functional / E2E: black-box from the user's perspective.

---

### pytest: fixtures and conftest

```python
# conftest.py — shared fixtures across all tests in the directory
import pytest
from django.test import Client

@pytest.fixture
def client():
    return Client()

@pytest.fixture
def user(db):                   # db — pytest-django fixture that wraps each test in a transaction
    from myapp.models import User
    return User.objects.create_user(username='alice', password='secret')

@pytest.fixture
def auth_client(client, user):  # fixtures can depend on other fixtures
    client.force_login(user)
    return client

# Scopes — how long the fixture lives:
@pytest.fixture(scope='session')   # once per test session (e.g. DB setup)
@pytest.fixture(scope='module')    # once per module
@pytest.fixture(scope='function')  # default — fresh per test
```

```python
# test_orders.py
def test_create_order(auth_client, user):
    response = auth_client.post('/api/orders/', {'item': 'book', 'qty': 2})
    assert response.status_code == 201
    assert response.json()['qty'] == 2
```

---

### parametrize

```python
import pytest

@pytest.mark.parametrize("input,expected", [
    (2, 4),
    (3, 9),
    (0, 0),
    (-2, 4),
])
def test_square(input, expected):
    assert input ** 2 == expected

# parametrize with IDs
@pytest.mark.parametrize("status,expected_code", [
    pytest.param('pending',   200, id='pending'),
    pytest.param('cancelled', 404, id='cancelled'),
])
def test_order_status(status, expected_code):
    ...
```

---

### Mocking

```python
from unittest.mock import patch, MagicMock, call

# patch as decorator — replaces object for the duration of the test
@patch('myapp.services.send_email')
def test_order_sends_email(mock_send_email):
    create_order(user_id=1, items=[...])
    mock_send_email.assert_called_once()
    mock_send_email.assert_called_with(user_id=1, template='order_confirmation')

# patch as context manager
def test_payment():
    with patch('myapp.payments.charge_card') as mock_charge:
        mock_charge.return_value = {'status': 'ok', 'charge_id': 'ch_123'}
        result = process_payment(amount=100)
    assert result['charge_id'] == 'ch_123'

# MagicMock — mock with magic methods (__len__, __iter__, etc.)
mock_qs = MagicMock()
mock_qs.__iter__.return_value = iter([order1, order2])
mock_qs.count.return_value = 2

# side_effect — raise an exception or return different values per call
mock_fn = MagicMock(side_effect=[1, 2, ValueError("boom")])
mock_fn()   # 1
mock_fn()   # 2
mock_fn()   # raises ValueError

# patch.object — patch a method on a specific object/class
with patch.object(MyService, 'fetch_data', return_value=[]) as mock:
    result = MyService().run()
```

---

### Factory Boy and model_bakery

```python
# factories.py — define once, reuse everywhere
import factory
from factory.django import DjangoModelFactory

class UserFactory(DjangoModelFactory):
    class Meta:
        model = 'auth.User'

    username = factory.Sequence(lambda n: f'user_{n}')
    email    = factory.LazyAttribute(lambda o: f'{o.username}@example.com')
    password = factory.PostGenerationMethodCall('set_password', 'password123')

class OrderFactory(DjangoModelFactory):
    class Meta:
        model = 'orders.Order'

    user   = factory.SubFactory(UserFactory)
    amount = factory.Faker('pydecimal', left_digits=4, right_digits=2, positive=True)
    status = 'pending'

# In tests:
user  = UserFactory()
order = OrderFactory(user=user, amount=99.99)
orders = OrderFactory.create_batch(10)   # create 10 at once

# model_bakery — simpler, auto-fills required fields
from model_bakery import baker

order = baker.make('orders.Order', status='pending')
orders = baker.make('orders.Order', _quantity=5)
```

---

### Property-based testing with Hypothesis

Tests with generated inputs — finds edge cases you wouldn't think to write:

```python
from hypothesis import given, settings, strategies as st

@given(st.lists(st.integers()))
def test_sort_is_idempotent(lst):
    assert sorted(sorted(lst)) == sorted(lst)

@given(st.text(min_size=1), st.integers(min_value=0, max_value=100))
def test_username_generation(name, number):
    result = generate_username(name, number)
    assert len(result) <= 50
    assert result.isalnum() or '_' in result

# Shrinking: Hypothesis automatically minimises failing inputs
# If [1, -5, 3, 100, -2] fails, it tries to find the smallest failing case: [-5]

@settings(max_examples=500)   # run more examples for critical code
@given(st.decimals(min_value=0, allow_nan=False))
def test_price_never_negative(price):
    result = apply_discount(price, discount=0.1)
    assert result >= 0
```

---

### Code quality tools

```bash
# Linting and formatting
black .                  # format (opinionated, no config needed)
isort .                  # sort imports
flake8 .                 # PEP8 + pyflakes
pylint myapp/            # deeper analysis, more rules

# Type checking
mypy myapp/ --strict     # static type checking

# Security
bandit -r myapp/         # find common security issues (hardcoded passwords, eval, etc.)

# Coverage
pytest --cov=myapp --cov-report=html
# open htmlcov/index.html
```

**pre-commit hooks** — run checks automatically before every commit:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.1.0
    hooks:
      - id: black
  - repo: https://github.com/pycqa/isort
    rev: 5.13.2
    hooks:
      - id: isort
  - repo: https://github.com/pycqa/flake8
    rev: 7.0.0
    hooks:
      - id: flake8
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
```

```bash
pre-commit install        # install hooks into .git/hooks/pre-commit
pre-commit run --all-files  # run manually on all files
```

---

### CSV Processing with Generators

```python
import csv

def process_large_csv(filepath: str):
    """Process large CSV without loading everything into memory."""
    with open(filepath, newline='') as f:
        reader = csv.DictReader(f)
        for row in reader:        # one row at a time
            yield process_row(row)

for result in process_large_csv('data.csv'):
    save_to_db(result)
```

---

### Django Service Layer Refactoring

```python
# Bad: view does everything
class OrderView(APIView):
    def post(self, request):
        order = Order.objects.create(user=request.user)
        for item in request.data['items']:
            OrderItem.objects.create(order=order, **item)
            inv = Inventory.objects.get(product_id=item['product_id'])
            inv.quantity -= item['quantity']
            inv.save()
        send_confirmation_email(request.user, order)
        return Response({'id': order.id})

# Good: SRP, bulk_create, atomic, on_commit
from django.db import transaction

class OrderService:
    @staticmethod
    def create_order(user, items: list) -> Order:
        with transaction.atomic():
            order = Order.objects.create(user=user)
            OrderItem.objects.bulk_create([
                OrderItem(order=order, **item) for item in items
            ])
            OrderService._update_inventory(items)
            transaction.on_commit(
                lambda: send_confirmation_email.delay(user.id, order.id)
            )
        return order

    @staticmethod
    def _update_inventory(items):
        product_ids = [i['product_id'] for i in items]
        inventories = {
            inv.product_id: inv
            for inv in Inventory.objects.select_for_update()
                                        .filter(product_id__in=product_ids)
        }
        for item in items:
            inventories[item['product_id']].quantity -= item['quantity']
        Inventory.objects.bulk_update(inventories.values(), ['quantity'])
```

---

---

### Mocking the database — when does it make sense?

**Yes — mocking the DB is worth it when DB-integrated tests are slow.**

Real DB tests (hitting PostgreSQL, MongoDB) typically take 10–100× longer than unit tests with mocked DB:
- Setting up fixtures, transactions, teardown
- Network/disk I/O even on localhost
- Schema migrations on test DB

```python
# Mock the repository layer — fast unit test, no DB needed
from unittest.mock import AsyncMock, MagicMock
import pytest
from services.order_service import OrderService

@pytest.mark.asyncio
async def test_create_order_insufficient_stock():
    order_repo = AsyncMock()
    product_repo = AsyncMock()
    product_repo.get.return_value = MagicMock(stock=2)  # only 2 in stock

    service = OrderService(order_repo, product_repo)

    with pytest.raises(InsufficientStockError):
        await service.create_order(user_id=1, product_id=5, qty=10)

    order_repo.create.assert_not_called()   # no order created
```

**The rule: mock at the boundary you own**

```
Endpoint tests  → mock Service
Service tests   → mock Repository
Repository tests → use real DB (these are integration tests)
```

**When to use a real DB (don't mock):**
- Testing the repository layer itself — SQL queries, ORM behavior, transactions
- Testing DB constraints (unique, FK, check constraints)
- Testing migrations
- End-to-end / integration tests

**Tools for fast DB tests when you need the real thing:**

```python
# pytest-django — wraps each test in a transaction, rolls back after
@pytest.mark.django_db(transaction=False)   # default: rollback after each test

# SQLite in-memory for unit tests (fast, no setup)
DATABASE_URL = "sqlite+aiosqlite:///:memory:"

# testcontainers — real Postgres in Docker, created per test session
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def postgres():
    with PostgresContainer("postgres:16") as pg:
        yield pg.get_connection_url()
```

**Summary:**

| Test type | DB strategy | Speed |
|---|---|---|
| Service / business logic | Mock repository | ⚡ Fast (ms) |
| Repository / ORM queries | Real DB (SQLite or testcontainers) | 🐢 Slower but necessary |
| API endpoint | Mock service or real DB + fixtures | Depends |
| Full integration / E2E | Real DB, real services | Slowest — run in CI only |

---
