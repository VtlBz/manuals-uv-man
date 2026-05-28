# Раздел 12. Миграция с существующих инструментов

---

!!! info "Альтернативный трек"
    Этот раздел не обязателен для базового greenfield-пути.
    Используйте его, если у вас уже есть существующий проект и нужно перейти на `uv`.

## Миграция с requirements.txt

Это самый распространенный сценарий. У вас есть проект с `requirements.txt`
(и, возможно, `requirements-dev.txt` и другие), и вы хотите перейти на `uv`.

### Исходная структура проекта

Типичный проект до миграции выглядит так:

```text
my-project/
├── app/
│   ├── __init__.py
│   └── main.py
├── tests/
│   └── test_main.py
├── requirements.txt
├── requirements-dev.txt
├── .python-version
└── README.md
```

Содержимое файлов зависимостей:

=== "requirements.txt"

    ```text
    flask==3.1.1
    sqlalchemy==2.0.40
    pydantic==2.11.1
    requests==2.32.3
    celery==5.5.2
    redis==5.3.0
    ```

=== "requirements-dev.txt"

    ```text
    pytest==8.3.5
    pytest-cov==6.1.1
    mypy==1.15.0
    ruff==0.11.8
    ```

### Шаг 1. Инициализация проекта

Если в проекте ещe нет `pyproject.toml`, выполните `uv init` в корневой директории:

```bash
cd my-project
uv init
```

`uv` создаст `pyproject.toml` с базовой структурой.

!!! warning "uv init и существующий `pyproject.toml`"
    Если файл `pyproject.toml` уже существует (например, с конфигурацией ruff
    или mypy), `uv init` завершится ошибкой. Варианты решения с примерами
    описаны в разделе [Интеграция uv в существующий проект](#интеграция-uv-в-существующий-проект).

### Шаг 2. Импорт основных зависимостей

Импортируйте зависимости из `requirements.txt`:

```bash
uv add -r requirements.txt
```

Эта команда:

- прочитает все пакеты из `requirements.txt`;
- добавит их в секцию `[project.dependencies]` в `pyproject.toml`;
- разрешит дерево зависимостей и создаст `uv.lock`;
- синхронизирует виртуальное окружение.

### Шаг 3. Импорт dev-зависимостей

Для dev-зависимостей используйте флаг `--dev`:

```bash
uv add --dev -r requirements-dev.txt
```

Dev-зависимости попадут в секцию `[dependency-groups]`:

```toml
[dependency-groups]
dev = [
    "pytest>=9.0.3",
    "pytest-cov>=7.1.0",
    "mypy>=2.0.0",
    "ruff>=0.15.12",
]
```

!!! tip "Именованные группы зависимостей"
    Если у вас более сложная структура (например, отдельные файлы для тестов,
    документации, CI), создайте именованные группы:

    ```bash
    uv add --group test -r requirements-test.txt
    uv add --group docs -r requirements-docs.txt
    ```

### Шаг 4. Что будет с версиями

Если ваш `requirements.txt` содержит точные пины (`flask==3.1.1`), после
`uv add -r requirements.txt` они попадут в `pyproject.toml` как есть.
Функционально это работает, но идиоматичнее хранить в `pyproject.toml` гибкие
спецификаторы (`>=3.1`), а точную фиксацию оставлять `uv.lock`.

После импорта рекомендуется вручную ослабить версии в `pyproject.toml`:

```toml
[project]
dependencies = [
    "flask>=3.1",
    "sqlalchemy>=2.0",
]
```

Затем зафиксировать точные версии в lockfile:

```bash
uv lock
```

!!! note "Если у вас pip-tools"
    При миграции с pip-tools у вас есть `requirements.in` (мягкие спецификаторы)
    и `requirements.txt` (скомпилированные точные версии). Флаг `-c` (constraint)
    позволяет импортировать мягкие спецификаторы, сохранив точные версии в `uv.lock`:

    ```bash
    uv add -r requirements.in -c requirements.txt
    ```

    В `pyproject.toml` попадут спецификаторы из `.in`, а `uv.lock`
    зафиксирует версии из `.txt`.

!!! warning "Constraint работает только с точными пинами"
    Флаг `-c` полезен для фиксации версий, только если файл содержит
    точные пины (`flask==3.1.1`). Если в нем диапазоны (`flask>=3.0,<4.0`),
    `uv` разрешит зависимости в рамках этих диапазонов и выберет наиболее
    подходящую (обычно новейшую) версию - воспроизвести старое окружение
    в точности не получится.

### Шаг 5. Проверка

Убедитесь, что окружение синхронизировано и проект работает:

```bash
# Синхронизация окружения
uv sync

# Запуск тестов
uv run pytest

# Проверка запуска приложения
uv run python -m app.main
```

### Шаг 6. Удаление старых файлов

После успешной проверки удалите устаревшие файлы:

```bash
rm requirements.txt requirements-dev.txt
```

Если были файлы `requirements.in`, `constraints.txt` или аналогичные -
удалите и их.

### Шаг 7. Обновление инфраструктуры

Обновите все файлы, которые ссылаются на старые зависимости:

!!! info "Пререквизиты для шага"
    Перед обновлением Docker/CI убедитесь, что уже выполнены:
    - импорт зависимостей в `pyproject.toml`;
    - создание актуального `uv.lock`;
    - успешный локальный запуск через `uv sync` и `uv run`.

**1. Dockerfile** (сборка образа):

До миграции:

```dockerfile
FROM python:3.12-slim
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "-m", "app.main"]
```

После миграции:

```dockerfile
FROM python:3.12-slim
# Установка uv
COPY --from=ghcr.io/astral-sh/uv:0.11.14 /uv /uvx /bin/
# Копирование файлов проекта
COPY pyproject.toml uv.lock ./
# Установка зависимостей (без dev)
RUN uv sync --frozen --no-dev --no-editable
COPY . .
CMD ["uv", "run", "python", "-m", "app.main"]
```

Это минимально runnable вариант для миграции. Более сложные production-template
паттерны (multi-stage, cache mount, разделение слоев) - в разделе
[Docker и CI/CD](10-docker-ci.md).

!!! note ""
    `--no-dev` отключает только группу `dev`.
    Подробнее - в разделе [Группы зависимостей](06-dependencies.md#группы-зависимостей).

**2. CI/CD** (GitHub Actions):

До миграции:

```yaml
steps:
  - uses: actions/checkout@v6
  - uses: actions/setup-python@v6
    with:
      python-version: "3.12"
  - run: pip install -r requirements.txt -r requirements-dev.txt
  - run: pytest
```

После миграции:

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v6

  - name: Install uv
    uses: astral-sh/setup-uv@v8
    with:
      version: "0.11.14"
      enable-cache: true

  - name: Check lockfile
    run: uv lock --check

  - name: Install dependencies
    run: uv sync --frozen --all-extras --all-groups

  - name: Test
    run: uv run --frozen pytest
```

Это минимально runnable CI-вариант после миграции. Полный production-template
шаблон с линтингом, проверкой типов и матрицей
версий - в разделе [Docker и CI/CD](10-docker-ci.md#github-actions).

**3. README.md** (инструкция для разработчиков):

До миграции:

```markdown
## Setup
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pip install -r requirements-dev.txt
```

После миграции:

```markdown
## Setup
uv sync
```

### Полный пример: до и после

=== "До миграции"

    ```text
    my-project/
    ├── app/
    ├── tests/
    ├── requirements.txt          # Production dependencies (pinned)
    ├── requirements-dev.txt      # Dev/test dependencies (pinned)
    ├── .python-version           # pyenv version file
    ├── Dockerfile
    └── README.md
    ```

    Установка окружения:

    ```bash
    pyenv install 3.12.4
    pyenv local 3.12.4
    python -m venv .venv
    source .venv/bin/activate
    pip install -r requirements.txt
    pip install -r requirements-dev.txt
    ```

=== "После миграции"

    ```text
    my-project/
    ├── app/
    ├── tests/
    ├── pyproject.toml            # Project metadata + dependencies
    ├── uv.lock                   # Universal lockfile (auto-generated)
    ├── .python-version           # Works with both pyenv and uv
    ├── Dockerfile
    └── README.md
    ```

    Установка окружения:

    ```bash
    uv sync
    ```

---

## Миграция с pip-tools

Если ваш проект использует `pip-tools` (`pip-compile` + `pip-sync`),
миграция особенно проста - концепции практически идентичны.

### Таблица соответствия

| pip-tools | uv | Описание |
| --------- | -- | -------- |
| `requirements.in` | `[project.dependencies]` в pyproject.toml | Список прямых зависимостей |
| `requirements-dev.in` | `[dependency-groups.dev]` в pyproject.toml | Dev-зависимости |
| `pip-compile` | `uv lock` | Генерация lockfile из списка зависимостей |
| `pip-sync` | `uv sync` | Синхронизация окружения с lockfile |
| `requirements.txt` (lockfile) | `uv.lock` | Файл с зафиксированными версиями |
| `pip-compile --upgrade` | `uv lock --upgrade` | Обновление всех зависимостей |
| `pip-compile -P flask` | `uv lock --upgrade-package flask` | Обновление одного пакета |

### Пошаговый процесс

**Шаг 1.** Если `pyproject.toml` отсутствует, инициализируйте `uv`-проект:

```bash
uv init
```

**Шаг 2.** Импортируйте зависимости из `.in`-файлов с ограничениями из `.txt`-файлов:

```bash
# Импорт production-зависимостей с фиксацией версий
uv add -r requirements.in -c requirements.txt

# Импорт dev-зависимостей с фиксацией версий
uv add --dev -r requirements-dev.in -c requirements-dev.txt
```

**Шаг 3.** Проверьте, что lockfile зафиксировал ожидаемые версии:

```bash
# Показать дерево зависимостей
uv tree
```

**Шаг 4.** Синхронизируйте окружение и запустите тесты:

```bash
uv sync
uv run pytest
```

**Шаг 5.** Удалите старые файлы:

```bash
rm requirements.in requirements-dev.in
rm requirements.txt requirements-dev.txt
```

### Сохранение точных версий

Ключевой момент при миграции с pip-tools - сохранение точных версий пакетов.
Флаг `-c` (constraint) позволяет это сделать:

```bash
uv add -r requirements.in -c requirements.txt
```

Как это работает:

1. uv читает список пакетов из `requirements.in` (например, `flask`,
   `sqlalchemy>=2.0`);
2. при разрешении зависимостей `uv` учитывает ограничения из `requirements.txt`
   (например, `flask==3.1.1`);
3. в `pyproject.toml` попадают "мягкие" спецификаторы из `.in`-файла;
4. в `uv.lock` фиксируются те же точные версии, что были в `.txt`-файле.

!!! tip "Проверка после миграции"
    Сравните зафиксированные версии:

    ```bash
    # Export uv.lock to requirements.txt format for comparison
    uv export --no-hashes > /tmp/uv-versions.txt

    # Compare with the original lockfile
    diff requirements.txt /tmp/uv-versions.txt
    ```

    Незначительные расхождения в транзитивных зависимостях допустимы -
    резолверы `uv` и pip-tools могут выбирать разные (но совместимые) версии.

### Эквиваленты рабочих процессов

**1. pip-tools** (старый процесс):

```bash
# Add a new dependency
echo "httpx" >> requirements.in
pip-compile requirements.in -o requirements.txt
pip-sync requirements.txt

# Update all dependencies
pip-compile --upgrade requirements.in -o requirements.txt
pip-sync requirements.txt

# Update a specific package
pip-compile -P flask requirements.in -o requirements.txt
pip-sync requirements.txt
```

**2. uv** (новый процесс):

```bash
# Add a new dependency
uv add httpx

# Update all dependencies
uv lock --upgrade
uv sync

# Update a specific package
uv lock --upgrade-package flask
uv sync
```

---

## Миграция с pyenv

Переход с pyenv можно выполнять постепенно. Ниже описаны фазы -
от минимальных изменений до полного отказа от pyenv.

### Как uv обнаруживает pyenv-установленный Python

`uv` обнаруживает Python-интерпретаторы через `PATH`. Для pyenv:

- Если pyenv настроен через shims (`~/.pyenv/shims/` в `PATH`), `uv` находит Python через shims.
- `uv` также может найти pyenv-версии напрямую по полному пути.

### Проблема shims pyenv

Pyenv использует shims - скрипты-обертки в `~/.pyenv/shims/`. Если shim для версии существует, но сама версия не установлена, `uv` получит ошибку.

#### Решение 1: Активировать версии через pyenv global

```bash
pyenv global 3.11.11 3.12.9 3.13.2
```

#### Решение 2: Переменная PYENV_VERSION

```bash
PYENV_VERSION=3.12.9 uv venv
```

#### Решение 3: Полный путь к интерпретатору

```bash
uv venv --python ~/.pyenv/versions/3.12.9/bin/python
```

#### Решение 4: python-preference = managed

```bash
uv venv --python-preference managed --python 3.12
```

### Сравнение команд pyenv и uv

| Действие | pyenv | uv |
| -------- | ----- | -- |
| Установить версию | `pyenv install 3.12` | `uv python install 3.12` |
| Удалить версию | `pyenv uninstall 3.12.0` | `uv python uninstall 3.12` |
| Список установленных | `pyenv versions` | `uv python list --only-installed` |
| Закрепить для проекта | `pyenv local 3.12` | `uv python pin 3.12` |
| Закрепить глобально | `pyenv global 3.12` | `uv python pin --global 3.12` |
| Найти путь к Python | `pyenv which python3.12` | `uv python find 3.12` |

### Фаза 1. Используем pyenv для Python, uv для пакетов

На этом этапе вы продолжаете управлять версиями Python через pyenv, но
переходите на `uv` для установки пакетов и управления зависимостями.

Добавьте в конфигурацию проекта:

=== "pyproject.toml"

    ```toml
    [tool.uv]
    python-preference = "system"
    ```

=== "uv.toml"

    ```toml
    python-preference = "system"
    ```

Значение `system` говорит `uv`: "Используй Python, установленный в системе
(в данном случае - через pyenv), и не скачивай свой".

Рабочий процесс:

```bash
# Python по-прежнему управляется через pyenv
pyenv install 3.12.4
pyenv local 3.12.4

# Пакеты управляются через uv
uv sync
uv run pytest
```

### Фаза 2. Установка Python через uv параллельно с pyenv

На этом этапе вы начинаете использовать встроенное управление Python в uv,
но не удаляете pyenv.

```bash
# Установка Python через uv
uv python install 3.12

# Фиксация версии для проекта
uv python pin 3.12
```

В `pyproject.toml` можно изменить настройку или убрать её вовсе (по умолчанию
`uv` предпочитает managed-версии, но использует системные при необходимости):

=== "pyproject.toml"

    ```toml
    [tool.uv]
    # python-preference = "managed"  # default, can be omitted
    ```

=== "uv.toml"

    ```toml
    # python-preference = "managed"  # default, can be omitted
    ```

На этой фазе pyenv и `uv` сосуществуют.
pyenv по-прежнему доступен как запасной вариант.

### Фаза 3. Полный переход на uv для управления Python

Измените конфигурацию, чтобы `uv` использовал только собственные версии Python:

=== "pyproject.toml"

    ```toml
    [tool.uv]
    python-preference = "managed"
    ```

=== "uv.toml"

    ```toml
    python-preference = "managed"
    ```

Установите все необходимые версии через uv:

```bash
# Установка нужных версий
uv python install 3.11 3.12 3.13

# Проверка установленных версий
uv python list --only-installed
```

На этом этапе pyenv уже не участвует в рабочем процессе, но ещe установлен в системе.

### Фаза 4. Удаление pyenv (опционально)

Когда все проекты переведены на `uv` и вы убедились, что всё работает,
можно удалить pyenv. Рекомендуется поэтапный подход:

**Шаг 1. Отключить pyenv в конфигурации оболочки:**

```bash
# Закомментировать строки pyenv в ~/.bashrc или ~/.zshrc:
#   export PYENV_ROOT="$HOME/.pyenv"
#   export PATH="$PYENV_ROOT/bin:$PATH"
#   eval "$(pyenv init -)"
```

**Шаг 2. Проработать 1-2 недели без pyenv.**
Если возникнут проблемы, раскомментировать строки обратно.

**Шаг 3. Удалить pyenv:**

```bash
# Удаление pyenv (если установлен через git clone)
rm -rf ~/.pyenv
```

!!! warning "Не спешите удалять pyenv"
    Удаление pyenv - это необратимый шаг. Убедитесь, что **все** ваши проекты
    (включая редко используемые) работают с uv-управляемым Python. Фаза 4
    полностью опциональна - pyenv и `uv` могут сосуществовать без конфликтов.

### Файлы .python-version

Файл `.python-version` используется как pyenv, так и `uv`. Однако есть тонкости:

```bash
# pyenv format (patch version required)
echo "3.12.4" > .python-version

# uv format (minor version is enough)
uv python pin 3.12
# Creates .python-version with content: 3.12
```

`uv` интерпретирует `.python-version` гибко:

- `3.12` - любая версия 3.12.x;
- `3.12.4` - точная версия 3.12.4.

pyenv требует точную версию (с patch). Если вы на фазе 1-2, используйте полную
версию для совместимости:

```bash
uv python pin 3.12.4
```

### Миграция с pyenv-virtualenv

Если вы используете `pyenv-virtualenv` для управления виртуальными окружениями:

| pyenv-virtualenv | uv | Описание |
| ---------------- | -- | -------- |
| `pyenv virtualenv 3.12 my-env` | `uv venv` (автоматически) | Создание окружения |
| `pyenv activate my-env` | Не требуется | Активация (uv run делает это автоматически) |
| `pyenv deactivate` | Не требуется | Деактивация |
| `pyenv virtualenvs` | - | Список окружений |

!!! note "uv и виртуальные окружения"
    `uv` создает виртуальное окружение автоматически при первом `uv sync` или
    `uv run`. Окружение размещается в `.venv/` в корне проекта. Явное создание и
    активация окружения, как правило, не нужны.

### Конфликт с pyenv-virtualenv

Плагин pyenv-virtualenv записывает в `.python-version` имя виртуального окружения, а не номер версии. `uv` игнорирует такие записи и продолжает поиск интерпретатора по стандартному алгоритму.

---

## Миграция с Poetry

### Пошаговый план

1. Проверить, используется ли `[project]` (PEP 621) или только `[tool.poetry]`.
2. Перенести метаданные в `[project]` (PEP 621 формат).
3. Перенести runtime-зависимости в `[project.dependencies]`.
4. Перенести dev/test/lint/docs-зависимости в `[dependency-groups]`.
5. Перенести extras в `[project.optional-dependencies]`.
6. Перенести scripts в `[project.scripts]`.
7. Выбрать build backend (`hatchling`, `uv_build`, `setuptools`).
8. Выполнить `uv lock`.
9. Выполнить `uv sync` и прогнать тесты.
10. Удалить `poetry.lock` только после успешного CI.

!!! warning "Не конвертируйте lockfile"
    `poetry.lock` нельзя преобразовать в `uv.lock` напрямую.
    `uv.lock` должен быть построен самим `uv` через `uv lock`.

---

## Миграция с PDM

PDM часто уже использует PEP 621 (`[project]`), поэтому миграция
может быть проще. Основные шаги:

1. Проверить формат `pyproject.toml` - если `[project]` уже используется,
   метаданные переносить не нужно.
2. Перенести dev-зависимости из `[tool.pdm.dev-dependencies]` в `[dependency-groups]`.
3. Удалить `[tool.pdm]` секции.
4. Выполнить `uv lock` и `uv sync`.
5. Удалить `pdm.lock` после успешного CI.

---

## Миграция с Pipenv

1. Восстановить зависимости из `Pipfile` в `pyproject.toml`:
    - `[packages]` -> `[project.dependencies]`
    - `[dev-packages]` -> `[dependency-groups]` dev
2. Выполнить `uv lock` и `uv sync`.
3. Удалить `Pipfile` и `Pipfile.lock` после успешного CI.

!!! note "Общее правило миграции lockfile"
    Старые lock-файлы (`poetry.lock`, `pdm.lock`, `Pipfile.lock`) удаляются
    только после полной проверки нового workflow в CI.

---

## Автоматизированные инструменты миграции

### migrate-to-uv

Утилита `migrate-to-uv` автоматизирует процесс миграции с различных менеджеров
пакетов.

```bash
# Запуск без глобальной установки
uvx migrate-to-uv
```

Что делает `migrate-to-uv`:

- определяет текущий менеджер пакетов (pip, pip-tools, Poetry, Pipenv);
- конвертирует метаданные и зависимости в формат `pyproject.toml`;
- сохраняет версии пакетов;
- генерирует `uv.lock`;
- удаляет старые файлы конфигурации (опционально).

Поддерживаемые источники:

| Источник | Файлы, которые конвертируются |
| -------- | ----------------------------- |
| pip / requirements.txt | `requirements.txt`, `requirements-dev.txt` |
| pip-tools | `requirements.in`, `requirements-dev.in`, `*.txt` |
| Poetry | `pyproject.toml` (секция `[tool.poetry]`) |
| Pipenv | `Pipfile`, `Pipfile.lock` |

Пример использования:

```bash
# Пробный запуск - посмотреть, что изменится
uvx migrate-to-uv --dry-run

# Запуск миграции
uvx migrate-to-uv

# Сохранить старые файлы вместо удаления
uvx migrate-to-uv --keep-old-files
```

### Когда использовать автоматическую миграцию

| Сценарий | Рекомендация |
| -------- | ------------ |
| Простой проект с `requirements.txt` | Автоматическая или ручная - разница минимальна |
| Проект с pip-tools | Автоматическая миграция удобнее |
| Проект с Poetry/Pipenv | Автоматическая миграция значительно проще |
| Сложный проект с нестандартной структурой | Ручная миграция для полного контроля |
| Проект, где нужно сохранить точные версии | Ручная миграция с `-c` (constraint) |

!!! tip "Проверяйте результат"
    Независимо от способа миграции, всегда проверяйте результат:

    ```bash
    uv sync
    uv run pytest
    ```

---

## Интеграция uv в существующий проект

Частый сценарий: у команды уже есть проект с `pyproject.toml`, в котором
настроены Ruff и mypy, но зависимости управляются через `pip` +
`requirements.txt`. Как перейти на `uv`, не ломая существующие настройки?

!!! warning "`uv init` и существующий `pyproject.toml`"
    Если в директории уже есть `pyproject.toml`, команда `uv init` завершится
    ошибкой `"pyproject.toml already exists"`. `uv init` не умеет дополнять
    существующий файл - она создает новый с нуля.

Два способа интеграции:

**Вариант 1. Добавить секции вручную** (рекомендуется).
Дописать `[project]` и `[dependency-groups]` прямо в существующий
`pyproject.toml`. Настройки инструментов остаются на месте.

**Вариант 2. Через `uv init` + перенос настроек.**
Переименовать существующий файл, выполнить `uv init`, перенести
настройки инструментов из старого файла в новый:

```bash
mv pyproject.toml pyproject.toml.bak
uv init
# Затем перенести секции [tool.*] из pyproject.toml.bak в сгенерированный pyproject.toml
```

После этого импортировать зависимости по шагам [2](#шаг-2-импорт-основных-зависимостей)-[4](#шаг-4-что-будет-с-версиями)
из начала этого раздела (включая dev-группы, `.in`-файлы
и constraints при наличии).

Второй вариант проще, если в `pyproject.toml` мало настроек
инструментов, и удобнее начать с чистой структуры `uv`.

Ниже показан первый вариант подробно.

### Пример: до перехода

```toml
# pyproject.toml - существующая конфигурация (только настройки инструментов)
[tool.ruff]
line-length = 120
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B"]
ignore = ["E501"]

[tool.ruff.format]
quote-style = "double"

[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true

[tool.pytest.ini_options]
testpaths = ["tests"]
```

И рядом лежат файлы:

```text
requirements.in          # Direct dependencies
requirements.txt         # Pinned dependencies (pip-compile output)
requirements-dev.in      # Dev dependencies
requirements-dev.txt     # Pinned dev dependencies
```

### Пример: после перехода на uv

Добавляем секции `[project]` и
`[dependency-groups]` в существующий `pyproject.toml`:

```toml
# pyproject.toml - после интеграции uv
[project]
name = "myapp"
version = "1.0.0"
description = "Our existing application"
requires-python = ">=3.12"
dependencies = [
    "fastapi[standard]>=0.136",
    "pydantic>=2.0",
    "sqlalchemy>=2.0",
    "httpx>=0.28",
    "celery>=5.6",
]

[dependency-groups]
dev = [
    "pytest>=9.0",
    "pytest-cov>=7.1",
    "pytest-asyncio>=1.3",
    "ruff>=0.15",
    "mypy>=2.0",
    "pre-commit>=4.6",
]

# --- Существующие настройки инструментов ниже (не изменены) ---

[tool.ruff]
line-length = 120
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B"]
ignore = ["E501"]

[tool.ruff.format]
quote-style = "double"

[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true

[tool.pytest.ini_options]
testpaths = ["tests"]
```

### Шаги интеграции

**1.** Добавьте секции `[project]` и `[dependency-groups]` в существующий
`pyproject.toml` вручную (`uv init` не модифицирует существующий файл).

**2.** Сгенерируйте lockfile и создайте окружение:

```bash
uv lock
uv sync
```

**3.** Проверьте, что инструменты работают:

```bash
uv run ruff check .
uv run mypy src/
```

!!! note "Что происходит с настройками инструментов"
    Секции `[tool.ruff]`, `[tool.mypy]`, `[tool.pytest.ini_options]`
    остаются нетронутыми. uv работает только со своими секциями
    (`[project]`, `[dependency-groups]`, `[tool.uv]`, `[build-system]`)
    и не изменяет чужие конфигурации.

В примере выше зависимости вписаны вручную. Если у вас есть
`requirements.txt`, импортируйте зависимости из него -
процедура описана в разделе
[Миграция с requirements.txt](#миграция-с-requirementstxt).

---

## Работа с существующим pyproject.toml

Многие команды уже используют `pyproject.toml` для конфигурации инструментов
(ruff, mypy, pytest) без использования его для управления зависимостями. В этом
случае миграция на `uv` особенно проста.

### Что уже есть

Типичный `pyproject.toml` до миграции:

```toml
[tool.ruff]
line-length = 120
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]

[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --tb=short"
```

### Что нужно добавить

Для интеграции с uv нужно добавить три секции: `[project]`,
`[dependency-groups]` и (опционально) `[tool.uv]`:

=== "pyproject.toml"

    ```toml
    [project]
    name = "my-project"
    version = "0.1.0"
    description = "My awesome project"
    readme = "README.md"
    requires-python = ">=3.12"
    dependencies = [
        "flask>=3.1.3",
        "sqlalchemy>=2.0.49",
        "pydantic>=2.13.4",
    ]

    [dependency-groups]
    dev = [
        "pytest>=9.0.3",
        "pytest-cov>=7.1.0",
        "mypy>=2.0.0",
        "ruff>=0.15.12",
    ]

    [tool.uv]
    python-preference = "managed"

    [tool.ruff]
    line-length = 120
    target-version = "py312"

    [tool.ruff.lint]
    select = ["E", "F", "I", "UP"]

    [tool.mypy]
    python_version = "3.12"
    strict = true
    warn_return_any = true

    [tool.pytest.ini_options]
    testpaths = ["tests"]
    addopts = "-v --tb=short"
    ```

=== "uv.toml"

    ```toml
    python-preference = "managed"
    ```

### Ключевые моменты

- Секции `[tool.ruff]`, `[tool.mypy]`, `[tool.pytest.ini_options]`
  **остаются нетронутыми**;
- uv читает только `[project]`, `[dependency-groups]`, `[build-system]`
  и `[tool.uv]`;
- конфликтов между секциями не возникает - каждый инструмент читает
  только свою секцию;
- порядок секций в файле не имеет значения для TOML, но рекомендуется
  размещать `[project]` в начале.

### Сценарии миграции

В зависимости от исходного состояния проекта выбирайте подходящий раздел:

**1. Нет `pyproject.toml`** - начните с [Миграция с requirements.txt](#миграция-с-requirementstxt).
`uv init` создаст `pyproject.toml`, после чего зависимости импортируются
через `uv add -r` (основные, dev, именованные группы).

**2. Есть `pyproject.toml` (только конфиг инструментов, зависимости
в `requirements.txt`)** - это самый распространенный случай. Два варианта
решения с примерами описаны в разделе [Интеграция uv в существующий проект](#интеграция-uv-в-существующий-проект).

**3. Полноценный `pyproject.toml` с `[project]` (Poetry, Hatch, PDM)** -
используйте `uvx migrate-to-uv` для автоматической миграции,
см. [Автоматизированные инструменты миграции](#автоматизированные-инструменты-миграции).

---
