# Курсовые материалы: 05-dependencies.md

!!! info "Цели модуля"
    После прохождения этого модуля вы сможете:

    - Добавлять, обновлять и удалять зависимости командами `uv add` /
      `uv remove`.
    - Организовывать зависимости по группам: production, dev, test, lint.
    - Управлять lockfile через `uv lock` и синхронизировать окружение через
      `uv sync`.
    - Анализировать дерево зависимостей с помощью `uv tree`.
    - Разрешать конфликты транзитивных зависимостей через overrides и
      constraints.

---

## Практическое задание

### Задание 1. Работа с зависимостями

1. Создайте новый проект:

    ```bash
    uv init --app dep-practice --python 3.12
    cd dep-practice
    ```

2. Добавьте production-зависимости:

    ```bash
    uv add "fastapi[standard]" pydantic
    ```

3. Проверьте, что `pyproject.toml` обновился:

    ```bash
    cat pyproject.toml
    ```

### Задание 2. Группы зависимостей

1. Добавьте dev-зависимости:

    ```bash
    uv add --dev pytest ruff
    ```

2. Создайте группу `test`:

    ```bash
    uv add --group test pytest-cov
    ```

3. Посмотрите на секции `[dependency-groups]` в `pyproject.toml`.

### Задание 3. Исследование дерева зависимостей

1. Выведите полное дерево:

    ```bash
    uv tree
    ```

2. Найдите, какие пакеты зависят от `anyio`:

    ```bash
    uv tree --invert --package anyio
    ```

3. Посмотрите зависимости конкретного пакета:

    ```bash
    uv tree --package fastapi
    ```

### Задание 4. Обновление зависимостей

1. Проверьте, есть ли обновления:

    ```bash
    uv lock --upgrade --dry-run
    ```

2. Обновите только один пакет:

    ```bash
    uv lock --upgrade-package pydantic
    ```

3. Синхронизируйте окружение с обновленным lockfile:

    ```bash
    uv sync
    ```

---

## Контрольные вопросы

??? question "В чем разница между `[dependency-groups]` и `[project.optional-dependencies]`?"
    `[dependency-groups]` (PEP 735) - группы для организации процесса
    разработки. Они используются разработчиками проекта (dev, test, lint, docs)
    и никогда не устанавливаются у конечных пользователей.

    `[project.optional-dependencies]` - extras, предназначенные для
    пользователей вашей библиотеки. Устанавливаются через
    `pip install mylib[extra]` или `uv add "mylib[extra]"`.

??? question "Что делает `uv sync --locked`?"
    Проверяет, что `uv.lock` актуален и соответствует `pyproject.toml`. Если
    разработчик изменил зависимости в `pyproject.toml`, но не выполнил
    `uv lock`, команда завершится с ошибкой. Это используется в CI для гарантии,
    что lockfile всегда актуален.

??? question "Как обновить одну зависимость, не обновляя все остальные?"
    Используйте команду `uv lock --upgrade-package <name>`:

    ```bash
    uv lock --upgrade-package requests
    uv sync
    ```

    Это пересчитает lockfile, обновив только `requests` и его транзитивные
    зависимости до максимально допустимых версий. Все остальные пакеты сохранят
    текущие версии.

??? question "Где объявляются production-зависимости, а где dev-зависимости?"
    Production-зависимости - в секции `[project]`, поле `dependencies`:

    ```toml
    [project]
    dependencies = ["fastapi>=0.115", "pydantic>=2.0"]
    ```

    Dev-зависимости - в секции `[dependency-groups]`:

    ```toml
    [dependency-groups]
    dev = ["pytest>=8.0", "ruff>=0.8"]
    ```

    При деплое в production используйте `uv sync --no-dev`, чтобы исключить
    dev-группу.

??? question "Что происходит при выполнении `uv add`? (три действия)"
    Команда `uv add` выполняет три операции за один вызов:

    1. **Обновляет `pyproject.toml`** - добавляет пакет в соответствующую секцию
       зависимостей.
    2. **Пересоздает `uv.lock`** - пересчитывает полное дерево зависимостей с
       учетом нового пакета.
    3. **Синхронизирует `.venv`** - устанавливает новый пакет и его транзитивные
       зависимости в виртуальное окружение.

    Это одна из ключевых особенностей uv: одна команда вместо цепочки
    `pip install` + ручное обновление `requirements.in` + `pip-compile` +
    `pip-sync`.

