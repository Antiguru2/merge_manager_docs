# Руководство разработчика

## ⚙️ Подготовка окружения
1. Установите зависимости проекта (см. корневой `req.txt`).
2. Активируйте виртуальное окружение и выполните `python manage.py migrate`.
3. Убедитесь, что `merge_manager` добавлен в `INSTALLED_APPS`.

## 🧩 Работа с профилями слияния
### Создание нового профиля
```python
MERGE_MANAGER_SETTINGS = {
    "PROFILES": [
        {
            "label": "company",
            "model": "main.Company",
            "fields": {
                "title": {"strategy": "prefer_target"},
                "inn": {"strategy": "prefer_non_null"},
                "phone": {"strategy": "concat", "metadata": {"separator": " / "}},
            },
            "soft_delete_field": "is_active",
        }
    ]
}
```
- Используйте строку `<app_label>.<ModelName>` или сам класс модели.
- `fields` принимает `FieldMergeRule` или словарь с аналогичными ключами.
- Параметр `metadata` доступен внутри стратегии: он попадает в `context['metadata']`, а также разворачивается в сам контекст (например, `metadata['separator']` будет доступен как `context['separator']`).
- Установите `hard_delete=True`, если после слияния нужно физически удалить донора (soft-delete в этом случае пропускается).

### Регистрация профилей кодом
Если требуется динамика, можно передать фабрику:
```python
def company_profiles():
    from merge_manager import MergeProfile, FieldMergeRule
    from main.models import Company

    return [
        MergeProfile(
            label="company",
            model=Company,
            fields={"title": FieldMergeRule(strategy="prefer_target")},
        )
    ]

MERGE_MANAGER_SETTINGS = {"PROFILES": [company_profiles]}
```

## 🧠 Кастомные стратегии
```python
# companies/merge_strategies.py
from merge_manager.services.strategies import _is_empty

def prefer_longer(target, donor, context):
    if _is_empty(target):
        return donor
    if _is_empty(donor):
        return target
    return donor if len(str(donor)) > len(str(target)) else target

FIELD_STRATEGIES = MERGE_MANAGER_SETTINGS.get("FIELD_STRATEGIES", {})
MERGE_MANAGER_SETTINGS["FIELD_STRATEGIES"] = {
    **FIELD_STRATEGIES,
    "prefer_longer": "companies.merge_strategies.prefer_longer",
}
```
После добавления стратегии она доступна по ключу `prefer_longer` в описании полей.

## 🔁 Хуки до/после слияния
```python
profile = MergeProfile(
    label="partner",
    model="crm.Partner",
    pre_merge_hooks=["crm.merge_hooks.log_start"],
    post_merge_hooks=["crm.merge_hooks.notify"],
)
```
Хуки получают именованные параметры `target`, `donor`, `context`/`result`. Используйте их для валидации или интеграции с внешними сервисами.

## 🧪 Тестирование
- Локальные тесты: `python manage.py test merge_manager`
- При написании тестов на конкретные модели создавайте фикстуры и используйте `MergeService` напрямую.
- Для проверок dry-run сравнивайте исходные значения в БД.

## 🛠️ Отладка
- Включите dry-run: `service.merge(..., dry_run=True)`
- Изучите `result.changed_fields` и `result.relations`
- Проверяйте журнал `MergeOperation` в админке; поле `summary` хранит детальный JSON.

## 🔮 Расширение
- Планируется добавить REST API и UI для выбора кандидатов.
- Закладывайте архитектуру своих интеграций с учётом будущих batch‑операций.
