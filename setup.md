# Установка и настройка merge_manager

## 📋 Предпосылки
- Python 3.10+
- Django 4.2 (совместимо с остальными приложениями Cargo Calc)
- Активные приложения `django.contrib.contenttypes`, `django.contrib.admin`
- Настроенная база данных (SQLite подойдёт для разработки)

## 🚀 Установка внутри проекта Cargo Calc

### 1. Установите зависимости проекта
```bash
pip install -r req.txt
```

### 2. Подключите приложение
```python
# cargo_calc/settings.py
INSTALLED_APPS = [
    # ...
    'merge_manager',
]
```

### 3. Примените миграции
```bash
python manage.py migrate merge_manager
```

### 4. Добавьте конфигурацию
```python
MERGE_MANAGER_SETTINGS = {
    "PROFILES": [],  # заполните профилями (см. ниже)
    "DRY_RUN_DEFAULT": False,
    "SOFT_DELETE_FIELD": "is_active",
    "SOFT_DELETE_VALUE": False,
}
```

## ⚙️ Настройка профилей
Минимальный профиль для модели `main.City`:
```python
MERGE_MANAGER_SETTINGS["PROFILES"].append({
    "label": "city",
    "model": "main.City",
    "fields": {
        "name": {"strategy": "prefer_target"},
        "code": {"strategy": "prefer_non_null"},
        "population": {"strategy": "prefer_donor"},
    },
})
```

### Доступные ключи в профиле
| Ключ | Обязательный | Описание |
|------|--------------|----------|
| `label` | ✅ | Уникальное имя профиля |
| `model` | ✅ | Django модель или путь `app.Model` |
| `fields` | ✅ | Правила по полям (`FieldMergeRule`) |
| `soft_delete_field` | ❌ | Имя поля для soft-delete (по умолчанию `is_active`) |
| `soft_delete_value` | ❌ | Значение, которое устанавливается донору |
| `hard_delete` | ❌ | При `True` донор удаляется из базы после слияния |
| `pre_merge_hooks` | ❌ | Список callables до слияния |
| `post_merge_hooks` | ❌ | Список callables после слияния |

### Поддержка пользовательского интерфейса
- Для отображения кнопки редактирования в интерфейсе модель может реализовать свойство или метод `admin_change_url`.
- Если свойство отсутствует, иконка не показывается, а поисковые ответы вернут `admin_change_url = null`.
- Желательно использовать mixin `AdminChangeButtonMixin` (см. `data_connector/submodules/base_content_objects/mixins.py`) либо добавить собственное свойство, возвращающее `reverse('admin:app_model_change', args=[pk])`.

## 🧰 Полезные переменные окружения
Специальных переменных модуль не требует. Используйте стандартные переменные Django для настройки БД и логирования.

## ✅ Проверка установки
```bash
python manage.py check
python manage.py test merge_manager
```
Если тесты проходят, а таблица `merge_manager_mergeoperation` появилась в БД, модуль готов к работе.

## ❌ Типичные проблемы
| Симптом | Причина | Решение |
|---------|---------|---------|
| `ModuleNotFoundError: merge_manager` | Приложение не в `INSTALLED_APPS` | Добавьте `merge_manager` и перезапустите сервер |
| `FieldDoesNotExist` при soft-delete | В модели нет указанного поля | Переопределите `soft_delete_field` или добавьте поле в модель |
| Стратегия не найдена | Опечатка в ключе `strategy` | Проверьте настройки и регистрация в `FIELD_STRATEGIES` |

## 📞 Поддержка
- Общие вопросы — канал `#cargo-calc-dev`
- Ошибки в миграциях — команда платформы
- Улучшения и идеи — создайте запись в [банке идей](ideas/README.md)
