---
name: building-web-features
description: Task-driven guide for web development: Django backend, templates, DRF APIs, and JavaScript integration
---

# Building Web Features

## Backend: Django Views & APIs

Build Django web views, REST APIs, and integrate with Evennia's game backend.

### Django App Configuration

**Always create an `apps.py` file** for contrib modules:

```python
 evennia/contrib/mycontrib/apps.py
from django.apps import AppConfig

class MyContribConfig(AppConfig):
    name = "evennia.contrib.mycontrib"
    label = "mycontrib"
    verbose_name = "My Contrib"
    default_auto_field = "django.db.models.BigAutoField"
```

**In settings.py**, reference the AppConfig class:
```python
INSTALLED_APPS += (
    'evennia.contrib.mycontrib.apps.MyContribConfig',  # ✓ Correct
)
```

### Lazy Imports for Typeclass Modules

**Problem**: Django raises `AppRegistryNotReady` when `__init__.py` imports typeclasses at module level.

**Solution**: Use PEP 562 lazy imports via `__getattr__`:

```python
 evennia/contrib/mycontrib/__init__.py

def __getattr__(name):
    """Lazy import to avoid loading models before Django apps are ready."""
    _LAZY_IMPORTS = {
        "MyTypeclass": "evennia.contrib.mycontrib.typeclasses",
        "MyCmdSet": "evennia.contrib.mycontrib.commands",
    }

    if name in _LAZY_IMPORTS:
        import importlib
        module_path = _LAZY_IMPORTS[name]
        module = importlib.import_module(module_path)
        attr = getattr(module, name)
        globals()[name] = attr  # Cache for future access
        return attr

    raise AttributeError(f"module {__name__!r} has no attribute {name!r}")

__all__ = ["MyTypeclass", "MyCmdSet"]
```

**Why**: Django imports the module during `django.setup()` without triggering model loading.

### SaverDict Serialization for APIs

**Critical**: Evennia attributes (`obj.db.*`) return `_SaverDict`/`_SaverList` which aren't JSON serializable.

```python
from evennia.utils.dbserialize import deserialize
from django.http import JsonResponse

 ✓ CORRECT - Use deserialize() for nested Saver* objects
data = deserialize(script.db.metrics or {})
return JsonResponse({"metrics": data})

 ✗ WRONG - TypeError: Object of type _SaverDict is not JSON serializable
return JsonResponse({"metrics": script.db.metrics})
```

**When to use `deserialize()`**:
1. Before passing data to `JsonResponse()` or `json.dumps()`
2. When sending data over network/APIs
3. When passing to external libraries expecting plain Python types

**When NOT to use**:
1. When keeping data within Evennia (internal method calls)
2. When storing back to `obj.db.*` (Evennia handles conversion)
3. For simple attribute access (no serialization)

**Why**: `dict()` only converts top level. `deserialize()` recursively converts all nested Saver* objects.

### DRF API Patterns

Evennia's REST API (`evennia/web/api/`) establishes patterns to follow.

#### ViewSet vs ModelViewSet

| Use Case | Base Class | Lookup Field | When to Use |
|----------|-----------|--------------|-------------|
| Registry/tag lookup | `ViewSet` | Custom key | AI assistant, tag-based systems |
| ORM CRUD | `ModelViewSet` | `pk` (default) | Standard database models |

**ViewSet example** (registry/tag lookup):

```python
from rest_framework.viewsets import ViewSet
from rest_framework.decorators import action

class MyViewSet(ViewSet):
    lookup_field = 'entity_key'
    lookup_value_regex = '[^/]+'

    def list(self, request):
        entities = get_from_registry()  # NOT ORM
        return Response(MySerializer(entities, many=True).data)

    @action(detail=True, methods=['post'])
    def custom_action(self, request, entity_key=None):
        return Response({"success": True})
```

**ModelViewSet example** (ORM access):

```python
from evennia.web.api.views import TypeclassViewSetMixin
from rest_framework.viewsets import ModelViewSet

class MyModelViewSet(TypeclassViewSetMixin, ModelViewSet):
    serializer_class = MySerializer
    queryset = MyModel.objects.all()
    permission_classes = [EvenniaPermission]
```

#### Serializer Pattern

**For Evennia typeclasses**:

```python
from evennia.web.api.serializers import TypeclassSerializerMixin

class MySerializer(TypeclassSerializerMixin, serializers.Serializer):
    # Auto-handles: attributes, tags, aliases
    key = serializers.CharField()
    typeclass = serializers.CharField()

    def to_representation(self, instance):
        data = super().to_representation(instance)
        # Use deserialize() for SaverDict attributes
        data['custom_data'] = deserialize(instance.db.custom_data or {})
        return data
```

#### Router Pattern

```python
 evennia/contrib/mycontrib/api/urls.py
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.trailing_slash = "/?"  # Evennia convention
router.register(r'myview', MyViewSet, basename='myview')
urlpatterns = router.urls
```

**Auto-generates**: `list()`, `create()`, `retrieve()`, `update()`, `destroy()`, `@action` methods

### URL Configuration

```python
 evennia/contrib/mycontrib/urls.py
from django.urls import path, include
from .api import urls as api_urls

urlpatterns = [
    path('api/', include(api_urls)),           # /mycontrib/api/
    path('dashboard/', views.dashboard_view),  # /mycontrib/dashboard/
]
```

**Evennia URL structure**:
- `/` → Website
- `/webclient/` → Webclient
- `/api/` → REST API (namespace="api")
- `/admin/` → Django admin (optional)

### Staff-Only Views

```python
from django.contrib.admin.views.decorators import staff_member_required
from django.http import JsonResponse

@staff_member_required
def analytics_view(request):
    """Requires user.is_staff = True."""
    return JsonResponse({"data": "sensitive"})
```

---

## Frontend: Templates & UI

Build web templates with Bootstrap 4.6, integrate with Django context, and organize JavaScript.

### Template System

**Always extend from** `evennia/web/templates/website/base.html`

#### Critical: Template Block Names

| Block Name    | Purpose                              | Common Mistake      |
|---------------|--------------------------------------|---------------------|
| `titleblock`  | Page title in `<title>` tag          | Using `title`       |
| `header_ext`  | Additional `<head>` content (CSS/JS) | Using `header_block`|
| `content`     | Main content area                    | -                   |
| `sidebar`     | Optional sidebar                     | -                   |
| `footer`      | Footer customization                 | -                   |
| `scripts`     | JavaScript at end of `<body>`        | Using `script`      |

**Example template**:

```django
{% extends "website/base.html" %}

{% block titleblock %}My Page{% endblock %}
{% block header_ext %}<link rel="stylesheet" href="{% static 'myapp/myapp.css' %}">{% endblock %}
{% block content %}<div class="container"><h1>My Content</h1></div>{% endblock %}
{% block scripts %}<script src="{% static 'myapp/myapp.js' %}"></script>{% endblock %}
```

**Error**: Wrong block names (`title` instead of `titleblock`, `header_block` instead of `header_ext`) will break CSS loading and page layout.

#### Auto-Available Context Variables

Evennia provides these **automatically** - **DO NOT pass manually** or you'll waste code duplicating what's already available:

| Category | Variables |
|----------|-----------|
| User & Character | `account`, `puppet` |
| Game Info | `game_name`, `game_slogan` |
| Feature Flags | `webclient_enabled`, `rest_api_enabled`, `register_enabled`, `freeze_world` |
| Connection Info | `telnet_ports`, `ssh_ports`, `websocket_port`, `websocket_client_url` |

### Bootstrap 4.6 Framework

**Evennia uses Bootstrap 4.6.0** (NOT Bootstrap 5)

- Already loaded via CDN in `base.html`
- **DO NOT** import different version or use Bootstrap 5 classes

**Common classes**: `.container`, `.row`, `.col-*`, `.card`, `.card-header`, `.card-body`, `.d-flex`, `.mb-*`, `.mt-*`

**Custom CSS slot**: `mygame/web/static/website/css/custom.css` (already linked)

**Official docs**: https://www.evennia.com/docs/latest/Components/Web-Bootstrap-Framework.html

### JavaScript Organization

| File | Purpose | When to Create |
|------|---------|----------------|
| `shared_utils.js` | Common utilities | Used in 2+ pages |
| `page_name.js` | Page-specific logic | Per page |

**Template pattern**:

```django
{% block scripts %}
<script src="{% static 'mycontrib/shared_utils.js' %}"></script>
<script src="{% static 'mycontrib/dashboard.js' %}"></script>
{% endblock %}
```

**Extract to shared_utils.js when**:
- Used in 2+ pages
- Pure utility (no page-specific DOM)
- Data fetching, formatters, validators

### Static File Organization

| Location | Purpose |
|----------|---------|
| `evennia/web/static/website/css/website.css` | Base Evennia styles |
| `evennia/web/static/website/css/custom.css` | Game customization slot |
| `mygame/web/static/website/` | Override static files |
| `evennia/contrib/mycontrib/static/mycontrib/` | Contrib static files |

**Logo replacement**: `mygame/web/static/website/images/logo.png`

---

## Data Flow Patterns

Handle data movement between backend, frontend, and APIs correctly.

### Backend → Frontend (Serialization)

```python
from evennia.utils.dbserialize import deserialize
from evennia.scripts.models import ScriptDB

def my_api_endpoint(request):
    # Tag search with type filtering
    tagged = search.search_tag("ai_assistant:main", category="system")
    scripts = [obj for obj in tagged if isinstance(obj, ScriptDB)]

    if not scripts:
        return JsonResponse({"error": "Not found"}, status=404)

    script = scripts[0]

    # Use deserialize() for proper JSON serialization
    metrics = deserialize(script.db.metrics or {})
    execution_log = deserialize(script.db.execution_log or [])

    return JsonResponse({
        "metrics": metrics,
        "recent_logs": execution_log[-50:],
    })
```

### Frontend → Backend (Validation)

**DRF serializer data structure validation**:

**Common error**: Returning dict wrapper instead of expected array causes `TypeError: goals.map is not a function` in JavaScript.

1. **Inspect first**: Check actual data structure with `deserialize()`
   ```python
   data = deserialize(script.db.current_goals or {})
   print(f"Type: {type(data)}, Keys: {data.keys()}")
   ```

2. **Document schema**: Add comment to serializer
   ```python
   class GoalsSerializer(serializers.Serializer):
       """Data structure: {"goals": [...], "version": 1}"""
       def to_representation(self, instance):
           goals_dict = deserialize(instance.db.current_goals or {})
           goals_list = goals_dict.get("goals", [])  # Extract array
           return {"active_goals": goals_list}
   ```

3. **Match frontend expectations**: Extract correct data shape
   ```python
   # ✗ WRONG - Returns dict wrapper
   return {"active_goals": goals_dict}

   # ✓ CORRECT - Extracts array
   return {"active_goals": goals_dict.get("goals", [])}
   ```

4. **Test data shape**:
   ```python
   def test_goals_returns_array(self):
       response = self.client.get(url)
       self.assertIsInstance(response.data['active_goals'], list)
   ```

---

## Website View Mixins

For non-API views, use Evennia's view mixins:

```python
from evennia.web.website.views.mixins import TypeclassMixin, EvenniaDetailView

class MyDetailView(TypeclassMixin, EvenniaDetailView):
    model = MyTypeclass
    template_name = "myapp/detail.html"
    # Auto-generates page_title
```

**Available**: `TypeclassMixin`, `EvenniaCreateView`, `EvenniaDetailView`, `EvenniaUpdateView`, `EvenniaDeleteView`

---

## Testing Web Features

```python
from django.test import Client, TestCase

class TestMyAPI(TestCase):
    def setUp(self):
        self.client = Client()
        # Create test data

    def test_api_returns_correct_structure(self):
        response = self.client.get('/mycontrib/api/data/')
        self.assertEqual(response.status_code, 200)
        self.assertIsInstance(response.json()['active_goals'], list)
```

---

## Architecture Doc
- [Bootstrap](https://www.evennia.com/docs/latest/Components/Web-Bootstrap-Framework.html)

## API References
- [Web API](https://www.evennia.com/docs/latest/Components/Web-API.html)
