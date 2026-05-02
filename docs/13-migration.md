# Раздел 13. Миграция с существующих инструментов

---

## Миграция с requirements.txt

Это самый распространенный сценарий. У вас есть проект с `requirements.txt` (и,
возможно, `requirements-dev.txt`), и вы хотите перейти на uv.

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

Если в проекте ещe нет `pyproject.toml`, выполните `uv init` в корневой
директории:

```bash
cd my-project
uv init
```

uv создаст `pyproject.toml` с базовой структурой. Если файл `pyproject.toml` уже
существует (например, с конфигурацией ruff или mypy), добавьте в него секцию
`[project]` вручную или используйте `uv init` - он дополнит существующий файл.

!!! note "uv init и существующие файлы"
    Команда `uv init` не перезаписывает существующий `pyproject.toml`. Если файл
    уже есть, uv добавит недостающие секции (`[project]`), оставив все остальные
    настройки нетронутыми.

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
    "pytest>=8.3.5",
    "pytest-cov>=6.1.1",
    "mypy>=1.15.0",
    "ruff>=0.11.8",
]
```

!!! tip "Именованные группы зависимостей"
    Если у вас более сложная структура (например, отдельные файлы для тестов,
    документации, CI), создайте именованные группы:

    ```bash
    uv add --group test -r requirements-test.txt
    uv add --group docs -r requirements-docs.txt
    ```

### Шаг 4. Сохранение точных версий из lockfile

Если вы хотите сохранить точные версии пакетов, зафиксированные в вашем текущем
`requirements.txt` (выступающем в роли lockfile), используйте флаг `-c`
(constraint):

```bash
uv add -r requirements.in -c requirements.txt
```

Здесь:

- `requirements.in` - файл с "мягкими" зависимостями (без версий или с
  диапазонами);
- `requirements.txt` - файл с точными версиями, используемый как constraint.

Это гарантирует, что `uv.lock` зафиксирует те же версии, которые были в вашем
`requirements.txt`, а в `pyproject.toml` попадут "мягкие" спецификаторы.

!!! warning "Порядок флагов имеет значение"
    Убедитесь, что файл с constraints (`-c`) содержит точные версии (с `==`).
    Если в `requirements.txt` есть диапазоны версий, они будут использованы как
    ограничения, но не как точные фиксации.

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

Если были файлы `requirements.in`, `constraints.txt` или аналогичные - удалите и
их.

### Шаг 7. Обновление инфраструктуры

Обновите все файлы, которые ссылаются на старые зависимости:

=== "Dockerfile"

    **До миграции:**

    ```dockerfile
    FROM python:3.12-slim
    COPY requirements.txt .
    RUN pip install -r requirements.txt
    COPY . .
    CMD ["python", "-m", "app.main"]
    ```

    **После миграции:**

    ```dockerfile
    FROM python:3.12-slim
    # Install uv
    COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
    # Copy project files
    COPY pyproject.toml uv.lock ./
    # Install dependencies (without dev)
    RUN uv sync --frozen --no-dev --no-editable
    COPY . .
    CMD ["uv", "run", "python", "-m", "app.main"]
    ```

=== "CI/CD (GitHub Actions)"

    **До миграции:**

    ```yaml
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: pytest
    ```

    **После миграции:**

    ```yaml
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv sync --frozen
      - run: uv run pytest
    ```

=== "README.md"

    **До миграции:**

    ```markdown
    ## Setup
    python -m venv .venv
    source .venv/bin/activate
    pip install -r requirements.txt
    pip install -r requirements-dev.txt
    ```

    **После миграции:**

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

Если ваш проект использует `pip-tools` (`pip-compile` + `pip-sync`), миграция
особенно проста - концепции практически идентичны.

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

**Шаг 1.** Инициализируйте uv-проект:

```bash
uv init
```

**Шаг 2.** Импортируйте зависимости из `.in`-файлов с ограничениями из
`.txt`-файлов:

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
2. при разрешении зависимостей uv учитывает ограничения из `requirements.txt`
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

    Незначительные расхождения в транзитивных зависимостях допустимы - резолверы
    uv и pip-tools могут выбирать разные (но совместимые) версии.

### Эквиваленты рабочих процессов

=== "pip-tools"

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

=== "uv"

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

Переход с pyenv можно выполнять постепенно. Ниже описаны четыре фазы - от
минимальных изменений до полного отказа от pyenv.

### Фаза 1. Используем pyenv для Python, uv для пакетов

На этом этапе вы продолжаете управлять версиями Python через pyenv, но
переходите на uv для установки пакетов и управления зависимостями.

Добавьте в `pyproject.toml`:

```toml
[tool.uv]
python-preference = "system"
```

Значение `system` говорит uv: "Используй Python, установленный в системе (в
данном случае - через pyenv), и не скачивай свой".

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

На этом этапе вы начинаете использовать встроенное управление Python в uv, но не
удаляете pyenv.

```bash
# Установка Python через uv
uv python install 3.12

# Фиксация версии для проекта
uv python pin 3.12
```

В `pyproject.toml` можно изменить настройку или убрать её вовсе (по умолчанию uv
предпочитает managed-версии, но использует системные при необходимости):

```toml
[tool.uv]
# Поведение по умолчанию: предпочитать uv-managed Python, фолбэк на системный
# python-preference = "managed"  # this is the default, can be omitted
```

На этой фазе pyenv и uv сосуществуют. pyenv по-прежнему доступен как запасной
вариант.

### Фаза 3. Полный переход на uv для управления Python

Измените конфигурацию, чтобы uv использовал только собственные версии Python:

```toml
[tool.uv]
python-preference = "managed"
```

Установите все необходимые версии через uv:

```bash
# Установка нужных версий
uv python install 3.11 3.12 3.13

# Проверка установленных версий
uv python list --only-installed
```

На этом этапе pyenv уже не участвует в рабочем процессе, но ещe установлен в
системе.

### Фаза 4. Удаление pyenv (опционально)

Когда все проекты переведены на uv и вы убедились, что всё работает:

```bash
# Удаление pyenv (если установлен через git clone)
rm -rf ~/.pyenv

# Удаление строк pyenv из конфига оболочки
# Edit ~/.bashrc or ~/.zshrc and remove:
#   export PYENV_ROOT="$HOME/.pyenv"
#   export PATH="$PYENV_ROOT/bin:$PATH"
#   eval "$(pyenv init -)"
```

!!! warning "Не спешите удалять pyenv"
    Удаление pyenv - это необратимый шаг. Убедитесь, что **все** ваши проекты
    (включая редко используемые) работают с uv-управляемым Python. Фаза 4
    полностью опциональна - pyenv и uv могут сосуществовать без конфликтов.

### Файлы .python-version

Файл `.python-version` используется как pyenv, так и uv. Однако есть тонкости:

```bash
# pyenv format (patch version required)
echo "3.12.4" > .python-version

# uv format (minor version is enough)
uv python pin 3.12
# Creates .python-version with content: 3.12
```

uv интерпретирует `.python-version` гибко:

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
    uv создает виртуальное окружение автоматически при первом `uv sync` или
    `uv run`. Окружение размещается в `.venv/` в корне проекта. Явное создание и
    активация окружения, как правило, не нужны.

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

## Работа с существующим pyproject.toml

Многие команды уже используют `pyproject.toml` для конфигурации инструментов
(ruff, mypy, pytest) без использования его для управления зависимостями. В этом
случае миграция на uv особенно проста.

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

```toml
# === NEW: project metadata (PEP 621) ===
[project]
name = "my-project"
version = "0.1.0"
description = "My awesome project"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "flask>=3.1.1",
    "sqlalchemy>=2.0.40",
    "pydantic>=2.11.1",
]

# === NEW: dependency groups ===
[dependency-groups]
dev = [
    "pytest>=8.3.5",
    "pytest-cov>=6.1.1",
    "mypy>=1.15.0",
    "ruff>=0.11.8",
]

# === NEW: uv-specific settings (optional) ===
[tool.uv]
python-preference = "managed"

# === EXISTING: untouched ===
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

### Ключевые моменты

- Секции `[tool.ruff]`, `[tool.mypy]`, `[tool.pytest.ini_options]` **остаются
  нетронутыми**;
- uv читает только `[project]`, `[dependency-groups]`, `[build-system]` и
  `[tool.uv]`;
- конфликтов между секциями не возникает - каждый инструмент читает только свою
  секцию;
- порядок секций в файле не имеет значения для TOML, но рекомендуется размещать
  `[project]` в начале.

!!! note "Автоматическое добавление секций"
    Команда `uv init` в директории с существующим `pyproject.toml` добавит
    только недостающие секции и не тронет остальные. Однако если вы хотите
    полный контроль над структурой файла, добавьте секции вручную.

---

## Чек-лист миграции

Используйте этот чек-лист при переводе проекта на uv:

- [ ] Инициализировать проект (`pyproject.toml` + `uv.lock`)
- [ ] Импортировать production-зависимости
- [ ] Импортировать dev/test-зависимости в соответствующие группы
- [ ] Убедиться, что `uv sync` выполняется без ошибок
- [ ] Запустить тесты через `uv run pytest`
- [ ] Обновить `Dockerfile` (если применимо)
- [ ] Обновить CI/CD-пайплайн (GitHub Actions, GitLab CI и т.д.)
- [ ] Обновить `README.md` (инструкции по установке)
- [ ] Обновить `.gitignore` (добавить `.venv/`, убедиться что `uv.lock` не в
  игноре)
- [ ] Удалить старые файлы (`requirements*.txt`, `Pipfile`, `Pipfile.lock` и
  т.д.)
- [ ] Уведомить команду и обновить документацию по онбордингу

!!! tip "Пошаговый подход"
    Не обязательно выполнять все пункты за один раз. Вы можете сначала выполнить
    миграцию зависимостей (пункты 1-5), убедиться что все работает, а затем
    обновить инфраструктуру (пункты 6-11) в отдельном коммите или PR.
