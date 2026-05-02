# Раздел 15. Best Practices и справочник

---

## Рекомендуемая структура проекта

```text
myproject/
  .python-version          # pinned Python version
  pyproject.toml           # project metadata and dependencies
  uv.lock                  # locked dependency versions
  README.md
  .gitignore
  src/
    myproject/
      __init__.py
      main.py
      ...
  tests/
    __init__.py
    test_main.py
    ...
  docs/
    ...
```

Такая структура (src-layout) рекомендуется для большинства проектов:

- **src-layout** предотвращает случайный импорт из исходников вместо
  установленного пакета;
- `pyproject.toml` - единая точка конфигурации проекта, зависимостей и
  инструментов;
- `uv.lock` фиксирует точные версии всех зависимостей;
- `.python-version` гарантирует единую версию Python для всех участников.

---

## Командные соглашения

При работе с uv в команде важно договориться о едином подходе. Ниже -
рекомендуемые соглашения.

### Что коммитить в git

| Файл / директория | Коммитить? | Почему |
| ----------------- | ---------- | ------ |
| `pyproject.toml` | Да | Основа конфигурации проекта |
| `uv.lock` | **Да** | Гарантирует воспроизводимые сборки |
| `.python-version` | Да | Единая версия Python в команде |
| `.venv/` | **Нет** | Генерируется автоматически через `uv sync` |
| `dist/` | Нет | Артефакты сборки |
| `__pycache__/` | Нет | Кеш Python |

!!! warning "uv.lock обязательно в git"
    Это самое важное правило. Без `uv.lock` в репозитории каждый разработчик и
    CI-сервер может получить разные версии зависимостей. Коммитите lockfile
    всегда.

### Правила для CI/CD

- Используйте `uv sync --locked` - падайте, если lockfile устарел.
- Используйте `uv run` для запуска тестов и скриптов - не активируйте окружение
  вручную.
- Кешируйте директорию кеша uv между сборками.

### Группы зависимостей

Договоритесь о единых именах групп:

```toml
[dependency-groups]
dev = ["ruff", "mypy", "pre-commit"]
test = ["pytest", "pytest-cov", "pytest-asyncio"]
docs = ["mkdocs-material", "mkdocstrings[python]"]
```

### Управление версией Python

- Зафиксируйте версию в `.python-version` через `uv python pin 3.12`.
- Установите `python-preference = "managed"` в `pyproject.toml` для
  единообразия:

```toml
[tool.uv]
python-preference = "managed"
```

Это заставит uv использовать свои управляемые версии Python, а не системные, что
устраняет проблему "у меня другая minor-версия".

---

## .gitignore для uv-проектов

```gitignore
# Виртуальное окружение
.venv/

# Кеш Python
__pycache__/
*.pyc
*.pyo

# Артефакты сборки
dist/
build/
*.egg-info/

# Переменные окружения (секреты)
.env
.env.local

# IDE
.idea/
.vscode/
*.swp
*.swo

# ОС
.DS_Store
Thumbs.db
```

!!! note "uv.lock НЕ добавляется в .gitignore"
    Обратите внимание: `uv.lock` отсутствует в списке. Этот файл **обязательно**
    должен быть в репозитории. Он обеспечивает воспроизводимость окружения для
    всех участников проекта и CI.

---

## Шпаргалка: ежедневные команды

### Управление проектом

| Задача | Команда |
| ------ | ------- |
| Создать новый проект | `uv init myproject` |
| Создать библиотеку | `uv init --lib mylib` |
| Синхронизировать окружение | `uv sync` |
| Синхронизировать (strict) | `uv sync --locked` |
| Синхронизировать без dev | `uv sync --no-dev` |

### Зависимости

| Задача | Команда |
| ------ | ------- |
| Добавить зависимость | `uv add requests` |
| Добавить с ограничением версии | `uv add "requests>=2.31"` |
| Добавить dev-зависимость | `uv add --dev pytest` |
| Добавить в группу | `uv add --group docs mkdocs-material` |
| Удалить зависимость | `uv remove requests` |
| Обновить lockfile | `uv lock` |
| Обновить все зависимости | `uv lock --upgrade` |
| Обновить конкретный пакет | `uv lock --upgrade-package requests` |
| Показать дерево зависимостей | `uv tree` |
| Показать обратные зависимости | `uv tree --invert` |

### Запуск кода

| Задача | Команда |
| ------ | ------- |
| Запустить скрипт | `uv run python app.py` |
| Запустить модуль | `uv run python -m myapp` |
| Запустить тесты | `uv run pytest` |
| Запустить с покрытием | `uv run pytest --cov` |
| Запустить инструмент (без установки) | `uvx ruff check .` |
| Запустить инструмент конкретной версии | `uvx ruff@0.9.0 check .` |

### Python

| Задача | Команда |
| ------ | ------- |
| Установить Python | `uv python install 3.12` |
| Установить несколько версий | `uv python install 3.11 3.12 3.13` |
| Зафиксировать версию | `uv python pin 3.12` |
| Список установленных версий | `uv python list` |
| Найти путь к Python | `uv python find 3.12` |

### Сборка и публикация

| Задача | Команда |
| ------ | ------- |
| Собрать пакет | `uv build` |
| Собрать только wheel | `uv build --wheel` |
| Собрать только sdist | `uv build --sdist` |
| Опубликовать на PyPI | `uv publish` |
| Опубликовать в приватный реестр | `uv publish --index company-registry` |

### Инструменты и утилиты

| Задача | Команда |
| ------ | ------- |
| Установить CLI-инструмент | `uv tool install ruff` |
| Список установленных инструментов | `uv tool list` |
| Обновить инструмент | `uv tool upgrade ruff` |
| Удалить инструмент | `uv tool uninstall ruff` |
| Обновить uv | `uv self update` |
| Очистить кеш | `uv cache clean` |
| Частично очистить кеш | `uv cache prune` |

---

## Таблица соответствия: старые инструменты -> uv

### pip -> uv

| Старая команда | Эквивалент uv | Примечание |
| -------------- | ------------- | ---------- |
| `pip install requests` | `uv add requests` | Проектный workflow (рекомендуется) |
| `pip install requests` | `uv pip install requests` | Pip-совместимый интерфейс |
| `pip install -r requirements.txt` | `uv pip install -r requirements.txt` | Или мигрируйте на `uv sync` |
| `pip install -e .` | `uv sync` | Editable-установка автоматическая |
| `pip uninstall requests` | `uv remove requests` | Проектный workflow |
| `pip uninstall requests` | `uv pip uninstall requests` | Pip-интерфейс |
| `pip freeze` | `uv pip freeze` | Список установленных пакетов |
| `pip show requests` | `uv pip show requests` | Информация о пакете |
| `pip list` | `uv pip list` | Список пакетов |

### pip-tools -> uv

| Старая команда | Эквивалент uv | Примечание |
| -------------- | ------------- | ---------- |
| `pip-compile requirements.in` | `uv pip compile requirements.in` | Pip-совместимый интерфейс |
| `pip-compile requirements.in` | `uv lock` | Проектный workflow |
| `pip-sync requirements.txt` | `uv pip sync requirements.txt` | Pip-совместимый интерфейс |
| `pip-sync requirements.txt` | `uv sync` | Проектный workflow |

### pyenv -> uv

| Старая команда | Эквивалент uv | Примечание |
| -------------- | ------------- | ---------- |
| `pyenv install 3.12` | `uv python install 3.12` | Установка Python |
| `pyenv local 3.12` | `uv python pin 3.12` | Записывает в `.python-version` |
| `pyenv global 3.12` | - | uv работает на уровне проекта |
| `pyenv versions` | `uv python list` | Список доступных версий |
| `pyenv which python` | `uv python find` | Путь к интерпретатору |

### virtualenv / venv -> uv

| Старая команда | Эквивалент uv | Примечание |
| -------------- | ------------- | ---------- |
| `python -m venv .venv` | `uv venv` | Явное создание |
| `python -m venv .venv` | (автоматически) | `uv sync` создает `.venv` при необходимости |
| `source .venv/bin/activate` | `uv run ...` | Активация не нужна |

### pipx -> uv

| Старая команда | Эквивалент uv | Примечание |
| -------------- | ------------- | ---------- |
| `pipx run ruff` | `uvx ruff` | Запуск без установки |
| `pipx install ruff` | `uv tool install ruff` | Глобальная установка |
| `pipx uninstall ruff` | `uv tool uninstall ruff` | Удаление |
| `pipx upgrade ruff` | `uv tool upgrade ruff` | Обновление |
| `pipx list` | `uv tool list` | Список инструментов |

### poetry -> uv

| Старая команда | Эквивалент uv | Примечание |
| -------------- | ------------- | ---------- |
| `poetry init` | `uv init` | Инициализация проекта |
| `poetry add requests` | `uv add requests` | Добавление зависимости |
| `poetry add --group dev pytest` | `uv add --dev pytest` | Dev-зависимость |
| `poetry remove requests` | `uv remove requests` | Удаление зависимости |
| `poetry install` | `uv sync` | Установка зависимостей |
| `poetry lock` | `uv lock` | Генерация lockfile |
| `poetry run pytest` | `uv run pytest` | Запуск команды |
| `poetry build` | `uv build` | Сборка пакета |
| `poetry publish` | `uv publish` | Публикация |
| `poetry show --tree` | `uv tree` | Дерево зависимостей |
| `poetry env use 3.12` | `uv python pin 3.12` | Выбор версии Python |

### twine -> uv

| Старая команда | Эквивалент uv | Примечание |
| -------------- | ------------- | ---------- |
| `twine upload dist/*` | `uv publish` | Публикация на PyPI |
| `twine upload -r private dist/*` | `uv publish --index private` | Приватный реестр |
| `twine check dist/*` | - | Проверка встроена в `uv build` |

---

## FAQ: частые ошибки и решения

### "No Python X.Y in PATH"

**Проблема:** uv не может найти запрошенную версию Python.

**Решение:**

```bash
# Установка нужной версии
uv python install 3.12

# Или, при использовании системного Python, убедитесь в его доступности
which python3.12
```

Если используете pyenv, убедитесь, что нужная версия установлена и доступна
через `pyenv global` или `pyenv local`.

---

### "Resolved lockfile is not up to date"

**Проблема:** `uv.lock` не соответствует `pyproject.toml`. Возникает при
использовании `--locked` (обычно в CI).

**Решение:**

```bash
# Перегенерация lockfile
uv lock

# Коммит обновленного lockfile
git add uv.lock
git commit -m "Update uv.lock"
```

Эта ошибка означает, что кто-то изменил зависимости в `pyproject.toml`, но не
выполнил `uv lock`.

---

### Ошибка парсинга .python-version (pyenv-virtualenv)

**Проблема:** uv не может разобрать файл `.python-version`, потому что в нем
записано имя виртуального окружения pyenv (например, `myproject-3.12`).

**Решение:** начиная с uv 0.6+, такие записи игнорируются. Если используете
более раннюю версию - обновите uv:

```bash
uv self update
```

Либо перезапишите `.python-version` чистой версией:

```bash
uv python pin 3.12
```

---

### "Package not found" при использовании приватного реестра

**Проблема:** uv не находит пакет, который есть в корпоративном реестре.

**Решение:** проверьте конфигурацию индексов в `pyproject.toml`:

```toml
[[tool.uv.index]]
name = "company-registry"
url = "https://pypi.company.com/simple/"
```

Убедитесь, что аутентификация настроена:

```bash
# Через переменные окружения
export UV_INDEX_COMPANY_REGISTRY_USERNAME="user"
export UV_INDEX_COMPANY_REGISTRY_PASSWORD="token"

# Или через keyring / netrc
```

---

### Медленный первый запуск

**Проблема:** первый `uv sync` или `uv add` работает заметно дольше последующих.

**Объяснение:** при первом запуске uv скачивает пакеты и заполняет локальный
кеш. Все последующие операции будут использовать кеш и работать значительно
быстрее. Это нормальное поведение.

---

### "Build backend not found"

**Проблема:** при попытке собрать пакет (`uv build`) или установить его в
editable-режиме возникает ошибка.

**Решение:** добавьте секцию `[build-system]` в `pyproject.toml`:

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

Эта секция необходима для библиотек. Приложения, созданные через `uv init`, не
имеют ее по умолчанию - это нормально, если вы не планируете собирать и
публиковать пакет.

---

### Кеш занимает слишком много места

**Проблема:** директория кеша uv выросла до значительного размера.

**Решение:**

```bash
# Удаление неиспользуемых записей кеша
uv cache prune

# Очистка всего кеша
uv cache clean

# Проверка расположения и размера кеша
uv cache dir
du -sh $(uv cache dir)
```

!!! tip "Регулярная очистка"
    Команда `uv cache prune` безопасна для регулярного использования - она
    удаляет только записи, которые больше не нужны. `uv cache clean` полностью
    очищает кеш - следующая установка будет скачивать пакеты заново.

---

### "Conflicting dependencies"

**Проблема:** uv не может разрешить конфликт версий между зависимостями.

**Решение:**

```bash
# Найти пакеты, вызывающие конфликт
uv tree --invert problematic-package

# Переопределить версию конкретной зависимости
```

Если конфликт нельзя разрешить штатными средствами, используйте override:

```toml
[tool.uv]
override-dependencies = [
    "problematic-package==1.5.0",
]
```

!!! warning "Override-dependencies"
    Переопределение версий зависимостей - крайняя мера. Используйте только когда
    нет другого способа разрешить конфликт, и тщательно тестируйте результат.

---

## Полезные ссылки

| Ресурс | Ссылка |
| ------ | ------ |
| Официальная документация | [docs.astral.sh/uv](https://docs.astral.sh/uv/) |
| GitHub-репозиторий | [github.com/astral-sh/uv](https://github.com/astral-sh/uv) |
| Руководство по миграции с pip | [Migration guide](https://docs.astral.sh/uv/guides/migration/pip-to-project/) |
| Справочник конфигурации | [Configuration reference](https://docs.astral.sh/uv/reference/settings/) |
| Справочник CLI | [CLI reference](https://docs.astral.sh/uv/reference/cli/) |

---

## Pip-совместимый интерфейс

uv предоставляет набор команд `uv pip ...`, которые являются прямой заменой для
pip и pip-tools.

### Доступные команды

```bash
# Прямая замена pip install
uv pip install requests flask

# Установка из файла requirements
uv pip install -r requirements.txt

# Компиляция requirements (аналог pip-compile)
uv pip compile requirements.in -o requirements.txt

# Синхронизация окружения по requirements (аналог pip-sync)
uv pip sync requirements.txt

# Показать установленные пакеты (аналог pip freeze)
uv pip freeze

# Удаление пакетов
uv pip uninstall requests

# Информация о пакете
uv pip show requests
```

### Когда использовать `uv pip` вместо проектного workflow

Pip-интерфейс полезен в следующих случаях:

- **Legacy-проекты** - проект использует `requirements.txt` и пока нет
  возможности мигрировать на `pyproject.toml`.
- **Постепенная миграция** - вы начинаете с замены pip на `uv pip` (мгновенное
  ускорение), а затем переходите на проектный workflow.
- **Одноразовые скрипты** - когда создание полного проекта избыточно.
- **CI/CD legacy-пайплайны** - замена `pip install` на `uv pip install` без
  рефакторинга всего пайплайна.

!!! note "Настройки pip-интерфейса"
    Секция `[tool.uv.pip]` в `pyproject.toml` или `uv.toml` влияет **только** на
    команды `uv pip ...`. Проектные команды (`uv sync`, `uv add`) используют
    собственные настройки из `[tool.uv]`.

---
