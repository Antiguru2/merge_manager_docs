# Структура директорий

```
merge_manager/
├── __init__.py              # Публичный API (переэкспорт профилей и реестра)
├── admin.py                 # Настройка Django Admin для журнала MergeOperation
├── apps.py                  # AppConfig + загрузка профилей при старте
├── config.py                # MergeProfile, FieldMergeRule и реестр
├── exceptions.py            # Специализированные исключения домена
├── migrations/
│   └── 0001_initial.py      # Миграция с таблицей merge_manager_mergeoperation
├── models.py                # Модель аудита MergeOperation
├── services/
│   ├── __init__.py          # Экспорт MergeService и MergeResult
│   ├── merge.py             # Основной сервис слияния и перенос связей
│   └── strategies.py        # Набор готовых стратегий по полям
├── settings.py              # Обёртка для MERGE_MANAGER_SETTINGS
├── tests.py                 # Интеграционные тесты движка
└── docs/                    # Документация подмодуля
    ├── README.md
    ├── overview.md
    ├── architecture/
    ├── guides/
    └── ...
```

Дополнительно:
- **`docs/architecture/`** — материалы по дизайну и принятым решениям.
- **`docs/guides/`** — инструкции для разработчиков и пользователей.
- **`docs/examples/`** — готовые сценарии интеграции.
