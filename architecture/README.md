# Архитектура merge_manager

## 🏗️ Общая архитектура
`merge_manager` построен по слоистой модели:
- **Configuration Layer** — декларативные профили и стратегии, считываемые на старте приложения.
- **Service Layer** — единый `MergeService`, управляющий процессом слияния, dry-run режимом и переносом связей.
- **Persistence Layer** — модель `MergeOperation` с использованием ContentType для универсальных связей.
- **Presentation Layer** — административный интерфейс для просмотра истории операций (UI для запуска слияний планируется отдельно).

## 📦 Основные компоненты
1. `settings.MergeManagerSettings` — загрузка и кэширование пользовательских настроек.
2. `config.MergeProfileRegistry` — ядро регистрации профилей, позволяющее расширять приложение без правки кода.
3. `services.merge.MergeService` — применение правил слияния, управление soft-delete, подготовка отчёта.
4. `services.strategies` — библиотека стратегий значений по умолчанию.
5. `models.MergeOperation` — журнал слияний с содержимым изменений.
6. `admin.MergeOperationAdmin` — удобный просмотр журнала и поиск по операциям.

## 🔄 Взаимодействия
```
Django settings  ──> MergeManagerConfig.ready() ──> load_profiles_from_settings()
                                             │
                                             ▼
                                    MergeProfileRegistry
                                             │
                                             ▼
                              MergeService.merge(target, donor, ...)
                                             │
                                             ├─ применяет стратегии полей
                                             ├─ переносит связи и soft-delete
                                             └─ пишет MergeOperation
```

## 📝 Архитектурные решения
Подробности и мотивация — в [decisions.md](decisions.md).
