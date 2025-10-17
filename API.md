# Документация API merge_manager

## 📋 Краткое описание

`merge_manager` предоставляет три типа интерфейсов для работы:

1. **Python API** — основной интерфейс через класс `MergeService`
2. **Конфигурационный API** — декларативная настройка через `MERGE_MANAGER_SETTINGS`
3. **Django Admin UI** — веб-интерфейс для просмотра журнала операций
4. **REST API** — *(в разработке)* HTTP API для интеграции с внешними системами

## 🐍 Python API

### MergeService

Основной класс для выполнения операций слияния записей.

#### Импорт

```python
from merge_manager.services import MergeService
```

#### Инициализация

```python
service = MergeService(profile_label: str)
```

**Параметры:**
- `profile_label` (string, required) — идентификатор профиля слияния из `MERGE_MANAGER_SETTINGS`

**Пример:**

```python
service = MergeService('city')
```

#### Метод `merge()`

Выполняет слияние двух записей согласно настроенному профилю.

```python
result = service.merge(
    target,
    donor,
    dry_run=False,
    field_overrides=None,
    extra_summary=None
)
```

**Параметры:**

- `target` (Model instance, required) — целевая запись (приёмник), которая останется после слияния
- `donor` (Model instance, required) — объединяемая запись (донор), которая будет отключена
- `dry_run` (bool, optional) — если `True`, выполняет предпросмотр без изменений в БД. По умолчанию берётся из настроек профиля
- `field_overrides` (dict, optional) — словарь переопределений стратегий для конкретных полей
- `extra_summary` (dict, optional) — дополнительные метаданные для сохранения в журнал

**Возвращает:**

Объект `MergeResult` с полями:
- `status` (string) — статус операции: `"completed"`, `"dry_run"`, или `"failed"`
- `changed_fields` (dict) — словарь изменений по полям: `{field_name: {"old": value, "new": value}}`
- `relations` (dict) — сводка перенесённых связей: `{"fk_count": int, "m2m_count": int}`
- `soft_delete` (dict) — информация о soft-delete донора
- `audit_record` (MergeOperation) — ссылка на созданную запись аудита (или `None` при dry_run)

**Пример базового использования:**

```python
from merge_manager.services import MergeService
from main.models import City

# Получаем две записи
city_primary = City.objects.get(id=1)
city_duplicate = City.objects.get(id=2)

# Создаём сервис
service = MergeService('city')

# Выполняем слияние
result = service.merge(target=city_primary, donor=city_duplicate)

# Проверяем результат
print(f"Статус: {result.status}")
print(f"Изменённые поля: {result.changed_fields}")
```

**Пример с dry-run:**

```python
# Предпросмотр изменений
result = service.merge(
    target=city_primary,
    donor=city_duplicate,
    dry_run=True
)

if result.status == "dry_run":
    print(f"Будут изменены поля: {result.changed_fields}")
    # Подтверждаем и выполняем реальное слияние
    result = service.merge(target=city_primary, donor=city_duplicate)
```

**Пример с переопределением стратегий:**

```python
result = service.merge(
    target=city_primary,
    donor=city_duplicate,
    field_overrides={
        "name": {"strategy": "prefer_donor"},       # Взять имя донора
        "description": {"strategy": "concat", "separator": " | "}  # Объединить описания
    }
)
```

**Пример с дополнительными метаданными:**

```python
result = service.merge(
    target=city_primary,
    donor=city_duplicate,
    extra_summary={
        "operator": "admin@example.com",
        "reason": "Duplicate found by fuzzy matching",
        "ticket_id": "TASK-123",
        "confidence_score": 0.95
    }
)

# Метаданные сохраняются в audit_record.summary
print(result.audit_record.summary)
```

#### Исключения

- `ProfileNotFound` — профиль с указанным label не найден в настройках
- `ValidationError` — модели target и donor не совпадают, или не принадлежат к модели профиля
- `FieldDoesNotExist` — указанное поле в стратегии не существует в модели
- `IntegrityError` — ошибка целостности данных при сохранении

### MergeResult

Объект результата операции слияния.

**Поля:**

```python
class MergeResult:
    status: str                      # "completed", "dry_run", "failed"
    changed_fields: dict             # {field: {"old": val, "new": val}}
    relations: dict                  # {"fk_count": int, "m2m_count": int}
    soft_delete: dict                # {"field": str, "value": any}
    audit_record: MergeOperation     # Запись аудита (или None при dry_run)
```

**Пример использования:**

```python
result = service.merge(target=city1, donor=city2)

if result.status == "completed":
    print(f"✅ Слияние завершено успешно")
    print(f"Изменено полей: {len(result.changed_fields)}")
    print(f"Перенесено FK связей: {result.relations['fk_count']}")
    print(f"Перенесено M2M связей: {result.relations['m2m_count']}")
    print(f"ID записи аудита: {result.audit_record.id}")
```

## ⚙️ Конфигурационный API

Настройка профилей слияния через Django settings.

### Структура MERGE_MANAGER_SETTINGS

```python
MERGE_MANAGER_SETTINGS = {
    "PROFILES": [
        {
            "label": str,                    # Уникальный идентификатор
            "model": str,                    # "app_label.ModelName"
            "fields": {
                "field_name": {
                    "strategy": str,         # Имя стратегии
                    **strategy_params        # Параметры стратегии
                }
            },
            "soft_delete_field": str,        # По умолчанию "is_active"
            "soft_delete_value": any,        # По умолчанию False
            "hard_delete": bool,             # По умолчанию False
            "pre_merge_hooks": list,         # Список callables
            "post_merge_hooks": list,        # Список callables
        }
    ],
    "DRY_RUN_DEFAULT": bool,                 # По умолчанию False
    "SOFT_DELETE_FIELD": str,                # По умолчанию "is_active"
    "SOFT_DELETE_VALUE": any,                # По умолчанию False
}
```

### Встроенные стратегии полей

#### `prefer_target`

Оставляет значение целевой записи (приёмника).

```python
"field_name": {"strategy": "prefer_target"}
```

**Использование:** Поля, значения которых не нужно изменять.

#### `prefer_donor`

Берёт значение донора (объединяемой записи).

```python
"field_name": {"strategy": "prefer_donor"}
```

**Использование:** Поля донора более актуальные или полные.

#### `prefer_non_null`

Выбирает непустое значение. Приоритет у приёмника.

```python
"field_name": {"strategy": "prefer_non_null"}
```

**Использование:** Заполнение пустых значений приёмника значениями донора.

#### `concat`

Объединяет строковые значения с разделителем.

```python
"field_name": {
    "strategy": "concat",
    "separator": " | "  # По умолчанию " "
}
```

**Использование:** Поля, где важно сохранить информацию из обеих записей (описания, теги).

### Пользовательские стратегии

Вы можете создать собственную стратегию:

```python
def custom_strategy(target_value, donor_value, field_name, **kwargs):
    """
    Пользовательская стратегия слияния.
    
    Args:
        target_value: Значение поля приёмника
        donor_value: Значение поля донора
        field_name: Имя поля
        **kwargs: Дополнительные параметры из конфигурации
    
    Returns:
        Итоговое значение для поля приёмника
    """
    if kwargs.get('condition') == 'longer':
        return target_value if len(target_value) > len(donor_value) else donor_value
    return target_value

# Использование в профиле
"field_name": {
    "strategy": custom_strategy,
    "condition": "longer"
}
```

### Хуки (Hooks)

Хуки позволяют добавить дополнительную логику до и после слияния.

#### Pre-merge хуки

Вызываются перед началом слияния:

```python
def validate_merge(target, donor, **kwargs):
    """Валидация перед слиянием."""
    if target.status == 'archived':
        raise ValueError("Cannot merge into archived record")

MERGE_MANAGER_SETTINGS["PROFILES"][0]["pre_merge_hooks"] = [
    validate_merge
]
```

#### Post-merge хуки

Вызываются после успешного слияния:

```python
def notify_users(target, donor, merge_result, **kwargs):
    """Уведомление после слияния."""
    send_notification(f"Records merged: {donor.id} -> {target.id}")

MERGE_MANAGER_SETTINGS["PROFILES"][0]["post_merge_hooks"] = [
    notify_users
]
```

## 📊 Модель MergeOperation

Django модель для хранения истории операций слияния.

### Поля модели

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | BigAutoField | Первичный ключ |
| `profile` | CharField | Label профиля слияния |
| `target_content_type` | ForeignKey(ContentType) | Тип модели приёмника |
| `target_object_id` | PositiveIntegerField | ID приёмника |
| `target` | GenericForeignKey | Ссылка на приёмника |
| `donor_content_type` | ForeignKey(ContentType) | Тип модели донора |
| `donor_object_id` | PositiveIntegerField | ID донора |
| `donor` | GenericForeignKey | Ссылка на донора |
| `status` | CharField | Статус: completed / dry_run / failed |
| `summary` | JSONField | Детали операции (changed_fields, relations, extra) |
| `created_at` | DateTimeField | Время создания записи |

### Запросы к модели

```python
from merge_manager.models import MergeOperation

# Все успешные операции
completed = MergeOperation.objects.filter(status='completed')

# Операции для конкретной модели
city_merges = MergeOperation.objects.filter(
    target_content_type__app_label='main',
    target_content_type__model='city'
)

# Последние 10 операций
recent = MergeOperation.objects.order_by('-created_at')[:10]

# Получение деталей
operation = MergeOperation.objects.first()
print(operation.summary['changed_fields'])
print(operation.summary['relations'])
```

## 🖥️ Django Admin UI

Django Admin предоставляет веб-интерфейс для просмотра журнала операций.

### Доступ

1. Авторизуйтесь в Django Admin: `http://localhost:8000/admin/`
2. Найдите раздел **«MERGE_MANAGER»**
3. Откройте **«Merge operations»**

### Возможности

- **Список операций** — все операции с фильтрами по статусу, дате, модели
- **Детали операции** — просмотр изменённых полей, перенесённых связей, метаданных
- **Фильтры** — по статусу (completed/dry_run/failed), дате, профилю
- **Поиск** — по ID приёмника или донора

## 🌐 REST API (в разработке)

> **Статус:** Планируется в будущих версиях

Планируемые эндпоинты:

### `GET /api/merge-manager/profiles/`

Получение списка доступных профилей слияния.

**Ответ:**

```json
{
  "profiles": [
    {
      "label": "city",
      "model": "main.City",
      "fields": {...}
    }
  ]
}
```

### `POST /api/merge-manager/merge/preview/`

Предпросмотр результата слияния (dry-run).

**Запрос:**

```json
{
  "profile": "city",
  "target_id": 1,
  "donor_id": 2
}
```

**Ответ:**

```json
{
  "status": "dry_run",
  "changed_fields": {
    "name": {"old": "Moscow", "new": "Moscow"}
  },
  "relations": {
    "fk_count": 5,
    "m2m_count": 2
  }
}
```

### `POST /api/merge-manager/merge/execute/`

Выполнение реального слияния.

**Запрос:**

```json
{
  "profile": "city",
  "target_id": 1,
  "donor_id": 2,
  "field_overrides": {...},
  "extra_summary": {...}
}
```

**Ответ:**

```json
{
  "status": "completed",
  "operation_id": 123,
  "changed_fields": {...},
  "relations": {...}
}
```

## 🔄 Интеграция с другими сервисами

### Celery задачи

Для асинхронного слияния можно обернуть вызов в Celery task:

```python
from celery import shared_task
from merge_manager.services import MergeService

@shared_task
def merge_cities_async(target_id, donor_id):
    from main.models import City
    
    target = City.objects.get(id=target_id)
    donor = City.objects.get(id=donor_id)
    
    service = MergeService('city')
    result = service.merge(target=target, donor=donor)
    
    return {
        "status": result.status,
        "operation_id": result.audit_record.id
    }
```

### Django Admin Actions

Добавьте действие слияния в Django Admin:

```python
from django.contrib import admin
from merge_manager.services import MergeService

@admin.register(City)
class CityAdmin(admin.ModelAdmin):
    actions = ['merge_selected']
    
    def merge_selected(self, request, queryset):
        if queryset.count() != 2:
            self.message_user(request, "Select exactly 2 records", level='error')
            return
        
        target, donor = queryset
        service = MergeService('city')
        result = service.merge(target=target, donor=donor)
        
        self.message_user(request, f"Merge {result.status}: {result.audit_record.id}")
    
    merge_selected.short_description = "Merge selected cities"
```

### REST API интеграция

Сохраняйте идентификаторы внешних систем через `extra_summary`:

```python
result = service.merge(
    target=target,
    donor=donor,
    extra_summary={
        "external_system_id": "EXT-12345",
        "source_api": "crm.example.com",
        "merge_confidence": 0.95
    }
)
```

## 📜 Версионирование

До появления публичного REST API версионирование делается через git-теги подмодуля.

Текущая версия: **1.1.0**

## 🔍 Примеры использования

### Пример 1: Простое слияние

```python
from merge_manager.services import MergeService
from main.models import City

service = MergeService('city')
result = service.merge(
    target=City.objects.get(name="Moscow"),
    donor=City.objects.get(name="Moskva")
)

print(f"✅ Слияние завершено. ID операции: {result.audit_record.id}")
```

### Пример 2: Слияние с предпросмотром

```python
service = MergeService('city')

# Шаг 1: Предпросмотр
preview = service.merge(target, donor, dry_run=True)
print(f"Будут изменены: {list(preview.changed_fields.keys())}")

# Шаг 2: Подтверждение
if input("Продолжить? (y/n): ") == 'y':
    result = service.merge(target, donor)
    print(f"✅ Выполнено")
```

### Пример 3: Массовое слияние

```python
from main.models import City

duplicates = [
    (City.objects.get(id=1), City.objects.get(id=2)),
    (City.objects.get(id=3), City.objects.get(id=4)),
]

service = MergeService('city')

for target, donor in duplicates:
    result = service.merge(target, donor)
    print(f"Merged {donor.name} -> {target.name}: {result.status}")
```

## 📞 Поддержка

При возникновении вопросов или проблем:

- **Общие вопросы:** канал `#cargo-calc-dev` (Slack)
- **Ошибки и баги:** создайте issue в workspace/tasks/tasks.md
- **Документация:** см. [OVERVIEW.md](OVERVIEW.md) и [QUICKSTART.md](QUICKSTART.md)

