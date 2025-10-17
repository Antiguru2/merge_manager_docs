# Быстрый старт с merge_manager

## 📋 Требования

### Минимальные зависимости

- **Python:** 3.10 или выше
- **Django:** 4.2 или выше (совместимо с остальными приложениями Cargo Calc)
- **База данных:** SQLite (для разработки) или PostgreSQL (для продакшена)

### Необходимые Django-приложения

В вашем проекте должны быть активированы:
- `django.contrib.contenttypes`
- `django.contrib.admin`

### Системные требования

- Настроенная база данных Django
- Доступ к Django Admin (для просмотра журнала операций)

## 🚀 Установка

### Шаг 1. Установите зависимости проекта

```bash
pip install -r req.txt
```

> **Примечание:** merge_manager использует стандартные библиотеки Django и не требует дополнительных зависимостей.

### Шаг 2. Подключите приложение

Добавьте `merge_manager` в `INSTALLED_APPS`:

```python
# cargo_calc/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    # ... другие приложения ...
    
    'merge_manager',  # Добавьте эту строку
]
```

### Шаг 3. Примените миграции

Для применения изменений в базе данных необходимо:
1. `python manage.py makemigrations merge_manager`
2. `python manage.py migrate merge_manager`

Это создаст таблицу `merge_manager_mergeoperation` для хранения журнала операций.

### Шаг 4. Добавьте конфигурацию

Добавьте в `settings.py` конфигурацию по умолчанию:

```python
# cargo_calc/settings.py
MERGE_MANAGER_SETTINGS = {
    "PROFILES": [],  # Будем заполнять профилями ниже
    "DRY_RUN_DEFAULT": False,
    "SOFT_DELETE_FIELD": "is_active",
    "SOFT_DELETE_VALUE": False,
}
```

## ⚙️ Настройка профилей слияния

### Минимальный профиль

Добавьте профиль для модели, которую хотите объединять. Например, для модели `main.City`:

```python
# cargo_calc/settings.py
MERGE_MANAGER_SETTINGS["PROFILES"].append({
    "label": "city",                    # Уникальный идентификатор профиля
    "model": "main.City",               # Путь к модели: app_label.ModelName
    "fields": {
        "name": {"strategy": "prefer_target"},      # Оставить имя приёмника
        "code": {"strategy": "prefer_non_null"},    # Взять непустое значение
        "population": {"strategy": "prefer_donor"}, # Взять население донора
    },
    "soft_delete_field": "is_active",   # Поле для мягкого удаления
    "soft_delete_value": False,         # Значение для отключения донора
})
```

### Доступные ключи в профиле

| Ключ | Обязательный | Описание |
|------|--------------|----------|
| `label` | ✅ | Уникальное имя профиля для обращения из кода |
| `model` | ✅ | Django модель в формате `"app_label.ModelName"` |
| `fields` | ✅ | Словарь правил слияния для каждого поля |
| `soft_delete_field` | ❌ | Имя поля для soft-delete (по умолчанию `is_active`) |
| `soft_delete_value` | ❌ | Значение, которое устанавливается донору (по умолчанию `False`) |
| `hard_delete` | ❌ | При `True` донор удаляется из базы после слияния |
| `pre_merge_hooks` | ❌ | Список callables, вызываемых до слияния |
| `post_merge_hooks` | ❌ | Список callables, вызываемых после слияния |

### Доступные стратегии полей

| Стратегия | Описание |
|-----------|----------|
| `prefer_target` | Оставить значение приёмника (целевой записи) |
| `prefer_donor` | Взять значение донора (объединяемой записи) |
| `prefer_non_null` | Взять непустое значение (приоритет у приёмника) |
| `concat` | Объединить строковые значения с разделителем |

**Пример стратегии concat:**

```python
"description": {
    "strategy": "concat",
    "separator": " | ",  # Разделитель между значениями
}
```

## 🎯 Первый запуск

### Базовый пример использования

Откройте Django shell и выполните слияние двух записей:

```bash
python manage.py shell
```

```python
from merge_manager.services import MergeService
from main.models import City

# Получите две записи для слияния
city_primary = City.objects.get(id=1)    # Приёмник (останется)
city_duplicate = City.objects.get(id=2)  # Донор (будет отключён)

# Создайте сервис слияния для профиля "city"
service = MergeService('city')

# Выполните слияние
result = service.merge(target=city_primary, donor=city_duplicate)

# Проверьте результат
print(f"Статус: {result.status}")
print(f"Изменённые поля: {result.changed_fields}")
print(f"Перенесённые связи: {result.relations}")
```

### Dry-run режим (предпросмотр)

Чтобы предварительно просмотреть результат без изменений в БД:

```python
# Выполните dry-run
result = service.merge(
    target=city_primary, 
    donor=city_duplicate, 
    dry_run=True  # Не вносить изменения в БД
)

# Проверьте, что будет изменено
print(f"Статус: {result.status}")  # dry_run
print(f"Будут изменены поля: {result.changed_fields}")
```

### Переопределение стратегий для конкретного случая

Вы можете переопределить стратегии для конкретной операции:

```python
result = service.merge(
    target=city_primary,
    donor=city_duplicate,
    field_overrides={
        "name": {"strategy": "prefer_donor"},  # Взять имя донора
    }
)
```

### Добавление дополнительных метаданных

Сохраните дополнительную информацию в журнал:

```python
result = service.merge(
    target=city_primary,
    donor=city_duplicate,
    extra_summary={
        "operator": "admin@example.com",
        "reason": "Duplicate found by fuzzy matching",
        "ticket_id": "TASK-123"
    }
)
```

## ✅ Проверка установки

### Проверка Django

Для применения изменений в базе данных необходимо:
1. `python manage.py check`

Если нет ошибок, значит приложение подключено корректно.

### Запуск тестов

Для применения изменений в базе данных необходимо:
1. `python manage.py test merge_manager`

Если тесты проходят успешно, модуль готов к работе.

### Проверка базы данных

Убедитесь, что таблица `merge_manager_mergeoperation` создана в базе данных.

### Проверка Django Admin

1. Авторизуйтесь в Django Admin: `http://localhost:8000/admin/`
2. Откройте раздел **«Merge operations»**
3. После первого слияния здесь появится запись с деталями операции

## ❌ Типичные проблемы

| Симптом | Причина | Решение |
|---------|---------|---------|
| `ModuleNotFoundError: merge_manager` | Приложение не в `INSTALLED_APPS` | Добавьте `'merge_manager'` в `INSTALLED_APPS` и перезапустите сервер |
| `FieldDoesNotExist` при soft-delete | В модели нет указанного поля | Переопределите `soft_delete_field` в профиле или добавьте поле `is_active` в модель |
| `ProfileNotFound` | Профиль с указанным label не найден | Проверьте `MERGE_MANAGER_SETTINGS["PROFILES"]` и label в коде |
| Стратегия не найдена | Опечатка в ключе `strategy` | Проверьте название стратегии в настройках профиля |
| Ошибка при миграциях | Конфликт зависимостей | Убедитесь, что `contenttypes` активирован в `INSTALLED_APPS` |

## 🎓 Что дальше?

После успешной установки и первого запуска:

- **[OVERVIEW.md](OVERVIEW.md)** — изучите архитектуру и принципы работы
- **[API.md](API.md)** — узнайте о всех возможностях MergeService
- **[CHANGELOG.md](CHANGELOG.md)** — ознакомьтесь с историей изменений

## 📞 Поддержка

- **Общие вопросы:** канал `#cargo-calc-dev` (Slack)
- **Ошибки в миграциях:** команда платформы
- **Улучшения и идеи:** создайте запись в workspace/tasks/tasks.md

## 💡 Полезные советы

### Поддержка кнопки редактирования в UI

Для отображения кнопки редактирования в интерфейсе модель может реализовать свойство или метод `admin_change_url`:

```python
from django.urls import reverse

class City(models.Model):
    # ... поля модели ...
    
    @property
    def admin_change_url(self):
        return reverse('admin:main_city_change', args=[self.pk])
```

Желательно использовать mixin `AdminChangeButtonMixin` (см. `data_connector/submodules/base_content_objects/mixins.py`) либо добавить собственное свойство.

### Логирование операций

Все операции слияния автоматически логируются в модель `MergeOperation`. Вы можете:
- Просматривать историю в Django Admin
- Фильтровать по моделям, статусам, датам
- Анализировать изменения через поле `summary`

