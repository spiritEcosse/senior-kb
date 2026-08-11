## 9. Тестирование и качество кода

### Виды тестирования

Unit: изолированная единица; без внешних зависимостей; максимальная скорость.
Integration: взаимодействие компонентов с реальными зависимостями.
Functional / E2E: чёрный ящик с точки зрения пользователя.

### Качество кода

- Одна функция = одна логическая операция (SRP)
- Чистые функции: результат только от аргументов, нет side-effects
- Вложенность: не более 3 уровней
- Цикломатическая сложность (mccabe): ≤ 8 путей
- Понятные имена

Инструменты: `flake8`, `pylint`, `black`, `isort`, `mypy`, `bandit` (безопасность), `pytest`, `coverage.py`.

### CSV с генераторами

```python
import csv

def process_large_csv(filepath: str):
    """Обработка большого CSV без загрузки всего в память."""
    with open(filepath, newline='') as f:
        reader = csv.DictReader(f)
        for row in reader:        # генератор: по одной строке
            yield process_row(row)

# Использование:
for result in process_large_csv('data.csv'):
    save_to_db(result)
```

`csv.DictReader` — каждая строка как словарь с именами из заголовка.

### Рефакторинг сервисного слоя Django

```python
# Плохо: всё в одном методе, несколько обязанностей
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

# Хорошо: SRP, bulk_create, atomic, on_commit
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
            # email отправляется только после успешного коммита
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

`select_for_update()` — блокировка строк на время транзакции (защита от race condition).
`bulk_create` / `bulk_update` — N операций в 1 SQL-запрос.

---
