# Базовый merge профиля

Этот пример показывает минимальную интеграцию профиля `city` и запуск слияния двух записей.

```python
from merge_manager import FieldMergeRule, MergeProfile, registry
from merge_manager.services import MergeService
from main.models import City

# Регистрация профиля (если не используется MERGE_MANAGER_SETTINGS)
profile = MergeProfile(
    label="city",
    model=City,
    fields={
        "name": FieldMergeRule(strategy="prefer_target"),
        "code": FieldMergeRule(strategy="prefer_non_null"),
        "population": FieldMergeRule(strategy="prefer_donor"),
    },
)
registry.register(profile)

service = MergeService("city")

target = City.objects.get(pk=1)
donor = City.objects.get(pk=42)

# Dry-run, чтобы посмотреть изменения
preview = service.merge(target=target, donor=donor, dry_run=True)
print(preview.changed_fields)

# Реальное слияние
result = service.merge(target=target, donor=donor)
print(result.status)
print(result.audit_record.summary)
```

После выполнения убедитесь, что запись появилась в админке в разделе «Merge operations».
