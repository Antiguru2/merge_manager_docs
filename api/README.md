# API и интеграции

## 🔌 Доступные интерфейсы
- **Python API** — класс `merge_manager.services.MergeService`.
- **Django Admin** — просмотр аудита операций.
- **Конфигурационный API** — настройка профилей через `MERGE_MANAGER_SETTINGS`.

## 🧰 Python API (MergeService)
```python
from merge_manager.services import MergeService

service = MergeService('city')
result = service.merge(target=target, donor=donor, dry_run=False)
```
- `target`, `donor` — экземпляры одной модели.
- `dry_run` — опциональный флаг, по умолчанию берётся из настроек.
- `field_overrides` — словарь ручных переопределений по полям.
- `extra_summary` — доп. данные, которые попадут в журнал.

`MergeResult` содержит:
- `status` (`completed` / `dry_run` / `failed`)
- `changed_fields` — словарь с историей изменений по полям
- `relations` — сводка перенесённых связей
- `soft_delete` — данные о soft-delete донора
- `audit_record` — ссылка на созданный `MergeOperation`

## 📡 HTTP / RPC API
На текущем этапе HTTP эндпоинты не предоставляются. Они зафиксированы в плане развития (см. `docs/plans/README.md`).

## 🔄 Интеграция с другими сервисами
- Для автоматизации добавьте вызов `MergeService` в админские действия или Celery-задачи.
- Используйте `extra_summary`, чтобы сохранять идентификаторы внешних систем или комментарии оператора.

## 📜 Версионирование
До появления публичного REST API версионирование делается через git-теги подмодуля.
