# Django REST Framework

## Serializers

Convert between Python objects (querysets/models) and JSON, and validate incoming data.

```python
from rest_framework import serializers

class OrderSerializer(serializers.ModelSerializer):
    total = serializers.SerializerMethodField()

    class Meta:
        model = Order
        fields = ['id', 'status', 'created_at', 'total']
        read_only_fields = ['id', 'created_at']

    def get_total(self, obj) -> float:
        return sum(item.price * item.qty for item in obj.items.all())

    def validate_status(self, value):
        if value not in dict(Order.STATUS_CHOICES):
            raise serializers.ValidationError('Invalid status')
        return value

    def validate(self, data):
        # cross-field validation
        if data.get('status') == 'shipped' and not data.get('tracking_number'):
            raise serializers.ValidationError('tracking_number required when shipped')
        return data
```

- `Serializer` — declare fields explicitly, full control, no model coupling.
- `ModelSerializer` — auto-generates fields from a model, `create()`/`update()` implemented for you.
- `validate_<field>` — single-field validation; `validate()` — cross-field validation.
- Nested serializers: `items = OrderItemSerializer(many=True, read_only=True)` — read-only by default because nested writes need explicit `create()`/`update()` overrides.

---

## Views: APIView vs Generics vs ViewSets

```python
# APIView — full manual control
class OrderDetail(APIView):
    def get(self, request, pk):
        order = get_object_or_404(Order, pk=pk, user=request.user)
        return Response(OrderSerializer(order).data)

# Generic view — CRUD boilerplate handled
class OrderDetail(generics.RetrieveUpdateDestroyAPIView):
    serializer_class = OrderSerializer

    def get_queryset(self):
        return Order.objects.filter(user=self.request.user)

# ViewSet — groups list/retrieve/create/update/destroy into one class
class OrderViewSet(viewsets.ModelViewSet):
    serializer_class = OrderSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Order.objects.filter(user=self.request.user)

    @action(detail=True, methods=['post'])
    def cancel(self, request, pk=None):
        order = self.get_object()
        order.status = 'cancelled'
        order.save(update_fields=['status'])
        return Response(OrderSerializer(order).data)
```

- `APIView` — one method per HTTP verb, no generated URLs; most explicit, most boilerplate.
- Generic views (`ListAPIView`, `RetrieveUpdateDestroyAPIView`, ...) — mixins for common patterns, still one route each.
- `ViewSet`/`ModelViewSet` — maps to a full resource; paired with a `Router` to auto-generate `list`, `create`, `retrieve`, `update`, `destroy` URLs. `@action` adds custom endpoints (`detail=True` → `/orders/{pk}/cancel/`, `detail=False` → `/orders/cancel/`).

---

## Routers

```python
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register('orders', OrderViewSet, basename='order')

urlpatterns = [
    path('api/', include(router.urls)),
]
```

`DefaultRouter` also exposes a browsable API root listing registered viewsets. `SimpleRouter` skips the root view.

---

## Authentication vs Permissions

Two separate steps: authentication identifies *who* is calling, permissions decide *what they can do*.

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}

# Custom permission — object-level check
class IsOwner(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.user == request.user

class OrderViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated, IsOwner]
```

- `has_permission` — runs before the view (e.g. is the user authenticated at all).
- `has_object_permission` — runs only for detail routes, after the object is fetched (ownership checks); generic list views never call it, which is why `get_queryset` must still filter by `request.user` (see IDOR note in [Web & API](web-api.md)).

---

## Throttling

```python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.UserRateThrottle',
        'rest_framework.throttling.AnonRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'user': '1000/day',
        'anon': '100/day',
    },
}

class LoginThrottle(AnonRateThrottle):
    scope = 'login'
    rate = '5/min'
```

Backed by Django's cache framework — same rate-limiting cache-key idea as the Redis approaches in [Caching](caching.md), just wired through DRF's throttle classes instead of a hand-rolled Lua script.

---

## Pagination

```python
class OrderPagination(PageNumberPagination):
    page_size = 25
    page_size_query_param = 'page_size'
    max_page_size = 100

REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'myapp.pagination.OrderPagination',
}
```

- `PageNumberPagination` — `?page=2`, simple, page count can shift under concurrent writes.
- `LimitOffsetPagination` — `?limit=25&offset=50`, SQL-`OFFSET`-like, same shifting risk on large offsets.
- `CursorPagination` — opaque cursor tied to an ordering field, stable under inserts/deletes; required when the underlying queryset changes between requests (e.g. a feed).

---

## Filtering

```python
# django-filter integration
class OrderFilter(filters.FilterSet):
    status = filters.CharFilter(field_name='status')
    created_after = filters.DateFilter(field_name='created_at', lookup_expr='gte')

    class Meta:
        model = Order
        fields = ['status', 'created_after']

class OrderViewSet(viewsets.ModelViewSet):
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_class = OrderFilter
    search_fields = ['id', 'user__email']
    ordering_fields = ['created_at', 'total']
```

`SearchFilter` → `?search=...` (uses `icontains` across `search_fields`). `OrderingFilter` → `?ordering=-created_at`.

---

## Versioning

```python
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
}

urlpatterns = [
    path('api/<str:version>/orders/', ...),
]

# in a view
def get_serializer_class(self):
    if self.request.version == 'v2':
        return OrderSerializerV2
    return OrderSerializer
```

`URLPathVersioning` (`/api/v1/...`), `NamespaceVersioning` (URL namespace), `AcceptHeaderVersioning` (`Accept: application/json; version=1.0`), `HostNameVersioning` (subdomain).

---

## Common pitfalls

- Nested writable serializers don't come for free — `ModelSerializer.create()`/`update()` only handle flat fields; nested `create()` must be overridden manually.
- `get_object_or_404` inside a `ViewSet.get_queryset()` filtered by `request.user` is the standard IDOR guard — permission classes alone don't filter querysets.
- `many=True` serializers instantiate one serializer per object — expensive `SerializerMethodField` logic (extra queries) multiplies per row; use `select_related`/`prefetch_related` in `get_queryset()` to avoid N+1.
- Browsable API renderer (`BrowsableAPIRenderer`) is convenient in dev but should be dropped from `DEFAULT_RENDERER_CLASSES` in production — it renders full HTML forms per request.
