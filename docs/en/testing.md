## 9. Testing and Code Quality

### Types of Testing

Unit: isolated unit; no external dependencies; maximum speed.
Integration: component interaction with real dependencies.
Functional / E2E: black-box from the user's perspective.

### Code Quality

- One function = one logical operation (SRP)
- Pure functions: result depends only on arguments, no side-effects
- Nesting: no more than 3 levels
- Cyclomatic complexity (mccabe): ≤ 8 paths
- Meaningful names

Tools: `flake8`, `pylint`, `black`, `isort`, `mypy`, `bandit` (security), `pytest`, `coverage.py`.

### CSV Processing with Generators

```python
import csv

def process_large_csv(filepath: str):
    """Process large CSV without loading everything into memory."""
    with open(filepath, newline='') as f:
        reader = csv.DictReader(f)
        for row in reader:        # generator: one row at a time
            yield process_row(row)

# Usage:
for result in process_large_csv('data.csv'):
    save_to_db(result)
```

`csv.DictReader` — each row as a dict with keys from the header row.

### Django Service Layer Refactoring

```python
# Bad: everything in one method, multiple responsibilities
class OrderView(APIView):
    def post(self, request):
        order = Order.objects.create(user=request.user)
        for item in request.data['items']:
            OrderItem.objects.create(order=order, **item)
            inventory = Inventory.objects.get(product_id=item['product_id'])
            inventory.quantity -= item['quantity']
            inventory.save()
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
            # email sent only after successful commit
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
        Inventory.objects.bulk_update(
            inventories.values(), ['quantity']
        )
```

`select_for_update()` — row-level lock for the duration of the transaction (prevents race conditions).
`bulk_create` / `bulk_update` — N operations in 1 SQL query.

---
