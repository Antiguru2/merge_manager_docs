# Быстрый старт с merge_manager

## 🚀 За 5 минут

### Шаг 1. Подключите приложение
```python
# cargo_calc/settings.py
INSTALLED_APPS = [
    # ...
    'merge_manager',
]
```

### Шаг 2. Примените миграции
```bash
python manage.py migrate merge_manager
```

### Шаг 3. Опишите профиль слияния
```python
# cargo_calc/settings.py
MERGE_MANAGER_SETTINGS = {
    "PROFILES": [
        {
            "label": "city",
            "model": "main.City",
            "fields": {
                "name": {"strategy": "prefer_target"},
                "code": {"strategy": "prefer_non_null"},
                "population": {"strategy": "prefer_donor"},
            },
            "soft_delete_field": "is_active",
            "soft_delete_value": False,
        }
    ],
}
```

### Шаг 4. Выполните слияние (Django shell)
```python
from merge_manager.services import MergeService

service = MergeService('city')
result = service.merge(target=city_primary, donor=city_duplicate)
print(result.changed_fields)
print(result.audit_record.summary)
```

### Шаг 5. Проверьте журнал в админке
1. Авторизуйтесь в Django Admin
2. Откройте раздел «Merge operations»
3. Убедитесь, что создана запись с ожидаемым статусом

## 🎯 Что дальше?
- [Полная установка и конфигурация](../setup.md)
- [Руководство разработчика](developer-guide.md)
- [Примеры использования](../examples/basic-usage.md)
