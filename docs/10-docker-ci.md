# Раздел 10. Docker и CI/CD

## uv в Docker

Использование uv в Docker-контейнерах дает заметный прирост скорости сборки
образов благодаря эффективному кешированию слоев и быстрой установке
зависимостей.

### Установка uv в Dockerfile

Рекомендуемый способ - копирование бинарного файла из официального образа:

```dockerfile
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
```

Этот способ быстрее, чем `pip install uv`, и не загрязняет зависимости проекта.

### Ключевые переменные окружения для Docker

| Переменная | Значение | Назначение |
| ---------- | -------- | ---------- |
| `UV_PYTHON_DOWNLOADS` | `never` | Использовать Python из контейнера, не скачивать |
| `UV_COMPILE_BYTECODE` | `1` | Компилировать .pyc-файлы для быстрого старта |
| `UV_LINK_MODE` | `copy` | Копировать файлы вместо hardlink (надежнее в Docker) |
| `UV_CACHE_DIR` | путь | Задать расположение кеша uv (полезно в CI) |

### Переменные окружения для Docker

Пример установки переменных в Dockerfile:

```dockerfile
ENV UV_LINK_MODE=copy \
    UV_COMPILE_BYTECODE=1 \
    UV_PYTHON_DOWNLOADS=never
```

Подробнее о каждой:

- **`UV_LINK_MODE=copy`** - по умолчанию `uv` пытается
  использовать hardlink (быстрее, экономит место),
  но в Docker это часто невозможно: cache mount
  и `.venv` находятся на разных файловых системах.
  С `copy` файлы просто копируются без предупреждений.
- **`UV_COMPILE_BYTECODE=1`** - после установки `uv`
  пре-компилирует `.py` в `.pyc`. Python не тратит
  время на компиляцию при первом импорте - старт
  приложения заметно ускоряется.
- **`UV_PYTHON_DOWNLOADS=never`** - запрещает `uv`
  скачивать managed Python. Если базовый образ уже
  содержит Python (`python:3.12-slim`), `uv` должен
  его использовать. Без этой переменной можно случайно
  получить два интерпретатора в образе.
- **`UV_CACHE_DIR`** - переопределяет путь к кешу `uv`
  (по умолчанию `~/.cache/uv`). В CI полезно указать
  каталог внутри проекта (например, `.cache/uv`),
  чтобы CI-система могла сохранять кеш между
  запусками.

### Оптимизация слоев

Ключевой прием - разделить копирование метаданных и кода:

1. Сначала скопировать `pyproject.toml` и `uv.lock`.
2. Выполнить `uv sync` - этот слой закешируется, пока зависимости не меняются.
3. Затем скопировать исходный код.

Это позволяет при изменении кода (без изменения зависимостей) использовать кеш
Docker для слоя с зависимостями.

### Базовый пример

```dockerfile
FROM python:3.12-slim

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

ENV UV_PYTHON_DOWNLOADS=never
ENV UV_COMPILE_BYTECODE=1
ENV UV_LINK_MODE=copy

WORKDIR /app

# Установка зависимостей (кешируемый слой)
COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-dev --no-install-project --no-cache

# Копирование кода приложения
COPY . .
RUN uv sync --locked --no-dev --no-cache

CMD ["uv", "run", "python", "-m", "myapp"]
```

!!! note "Флаг `--no-cache`"
    В Docker рекомендуется использовать `--no-cache`, чтобы uv не сохранял свой
    внутренний кеш в образе. Кеширование и так обеспечивается слоями Docker.

### Multi-stage build

Multi-stage build позволяет создать минимальный production-образ без uv и
инструментов сборки:

```dockerfile
# --- Этап 1: сборка ---
FROM ghcr.io/astral-sh/uv:latest AS uv

FROM python:3.12-slim AS builder

COPY --from=uv /uv /bin/uv

ENV UV_PYTHON_DOWNLOADS=never
ENV UV_COMPILE_BYTECODE=1
ENV UV_LINK_MODE=copy

WORKDIR /app

COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-dev --no-install-project --no-cache

COPY . .
RUN uv sync --locked --no-dev --no-cache

# --- Этап 2: runtime ---
FROM python:3.12-slim AS runtime

COPY --from=builder /app /app

WORKDIR /app

ENV PATH="/app/.venv/bin:$PATH"

CMD ["python", "-m", "myapp"]
```

В финальном образе нет ни uv, ни заголовочных файлов, ни кеша - только Python и
зависимости приложения.

### Production Dockerfile для FastAPI

Полный пример Dockerfile для FastAPI-приложения с оптимизацией:

```dockerfile
# --- Этап сборки ---
FROM ghcr.io/astral-sh/uv:latest AS uv

FROM python:3.12-slim AS builder

COPY --from=uv /uv /bin/uv

ENV UV_PYTHON_DOWNLOADS=never \
    UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy

WORKDIR /app

# Сначала установка зависимостей (кеширование слоев)
COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-dev --no-install-project --no-cache

# Копирование исходного кода и финализация установки
COPY src/ src/
COPY README.md ./
RUN uv sync --locked --no-dev --no-cache

# --- Этап runtime ---
FROM python:3.12-slim AS runtime

RUN groupadd --gid 1000 app \
    && useradd --uid 1000 --gid app --shell /bin/bash app

COPY --from=builder --chown=app:app /app /app

WORKDIR /app
USER app

ENV PATH="/app/.venv/bin:$PATH"

EXPOSE 8000

CMD ["uvicorn", "myapp.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

!!! tip "Непривилегированный пользователь"
    В production всегда запускайте приложение от непривилегированного
    пользователя. В примере создается пользователь `app` с UID 1000.

### Пример: multi-stage сборка

Расширенный вариант multi-stage build с `cache-mount`
и `bind-mount` для максимальной скорости повторных
сборок:

```dockerfile
# =========================================================
# Stage 1: builder - зависимости и проект
# =========================================================
FROM python:3.12-slim-bookworm AS builder

# uv нужен только на этапе сборки
COPY --from=ghcr.io/astral-sh/uv:0.5 /uv /uvx /bin/

ENV UV_LINK_MODE=copy \
    UV_COMPILE_BYTECODE=1 \
    UV_PYTHON_DOWNLOADS=never

WORKDIR /app

# Слой 1: зависимости (cache-mount + bind-mount)
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --no-install-project --no-dev

# Слой 2: код проекта
COPY . .
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

# =========================================================
# Stage 2: runtime - минимальный образ
# =========================================================
FROM python:3.12-slim-bookworm AS runtime

WORKDIR /app

# Только venv и код из builder
COPY --from=builder /app/.venv /app/.venv
COPY --from=builder /app/src /app/src

ENV PATH="/app/.venv/bin:$PATH"

# Непривилегированный пользователь
RUN useradd --create-home --shell /bin/bash app
USER app

CMD ["python", "-m", "myapp"]
```

Ключевые оптимизации в этом варианте:

- **`--mount=type=cache`** - кеш `uv` (скачанные
  wheels) сохраняется между сборками. При повторном
  билде `uv` не обращается к PyPI за уже скачанными
  пакетами.
- **`--mount=type=bind`** - `pyproject.toml` и
  `uv.lock` доступны только во время `RUN`, не
  создают отдельный слой в образе.
- **Два этапа `uv sync`** - первый устанавливает
  зависимости (кешируемый слой), второй - сам проект.
- **В runtime нет** `uv`, `uvx`, кеша wheels,
  заголовочных файлов - только Python, `.venv` и код.

!!! note "cache-mount в CI"
    В CI-системах cache-mount по умолчанию не
    сохраняется между запусками. Локально оптимизация
    работает автоматически, в CI требуется
    дополнительная настройка кеширования.

## uv в CI/CD

### Принципы использования uv в CI

1. **Всегда используйте `--locked`** - если lockfile устарел, сборка должна
   упасть, а не молча обновить зависимости.
2. **Не активируйте виртуальное окружение вручную** - используйте `uv run` для
   запуска команд.
3. **Кешируйте директорию uv** - это ускоряет повторные сборки.

### GitHub Actions

Astral предоставляет официальный action
`astral-sh/setup-uv`, который устанавливает `uv`
и настраивает кеширование.

**Полный workflow** (`.github/workflows/ci.yml`):

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Установка uv
        uses: astral-sh/setup-uv@v4
        with:
          enable-cache: true

      - name: Проверка lockfile
        run: uv lock --check

      - name: Установка зависимостей
        run: uv sync --locked --all-extras

      - name: Линтинг
        run: uvx ruff check .

      - name: Форматирование
        run: uvx ruff format --check .

      - name: Проверка типов
        run: uv run mypy src/

      - name: Тесты
        run: uv run pytest
```

Ключевые моменты:

- **`astral-sh/setup-uv@v4`** - устанавливает `uv`,
  автоматически кеширует `~/.cache/uv` между запусками
  при `enable-cache: true`.
- **`uv lock --check`** - отдельный шаг для проверки
  актуальности lockfile. Если разработчик изменил
  зависимости, но забыл обновить `uv.lock`, сборка
  упадет с понятным сообщением.
- **`uvx ruff`** - запускает `ruff` без установки
  в проект. Для `mypy` используется `uv run`, так
  как ему нужен доступ к зависимостям проекта.

**Матрица версий Python:**

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12", "3.13"]
    steps:
      - uses: actions/checkout@v4

      - uses: astral-sh/setup-uv@v4
        with:
          enable-cache: true

      - run: uv sync --locked --python ${{ matrix.python-version }}
      - run: uv run pytest
```

**Публикация пакета** (дополнительный job):

```yaml
  publish:
    needs: test
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv build
      - run: uv publish --trusted-publishing always
```

### GitLab CI

В GitLab CI нет аналога `astral-sh/setup-uv`,
поэтому `uv` устанавливается через `pip` или
используется Docker-образ Astral.

**Пример `.gitlab-ci.yml`:**

```yaml
default:
  image: python:3.12-slim

variables:
  UV_CACHE_DIR: .uv-cache
  UV_LINK_MODE: copy

cache:
  key:
    files:
      - uv.lock
      - pyproject.toml
  paths:
    - .uv-cache/
    - .venv/

stages:
  - check
  - test

check:lock:
  stage: check
  before_script:
    - pip install --no-cache-dir uv
  script:
    - uv lock --check

test:
  stage: test
  before_script:
    - pip install --no-cache-dir uv
    - uv sync --locked
  script:
    - uv run ruff check .
    - uv run pytest --junitxml=report.xml
  artifacts:
    reports:
      junit: report.xml

lint:
  stage: check
  before_script:
    - pip install --no-cache-dir uv
  script:
    - uvx ruff check .
    - uvx ruff format --check .
```

Ключевые моменты:

- **`UV_CACHE_DIR`** - перенаправляет кеш `uv`
  в каталог проекта (`.uv-cache/`), чтобы GitLab
  мог его сохранять через секцию `cache`.
- **Ключ кеша по файлам** - кеш привязан к содержимому
  `uv.lock` и `pyproject.toml`. При изменении
  зависимостей кеш пересоздается.
- **`UV_LINK_MODE=copy`** - аналогично Docker,
  копирует файлы вместо hardlink.

**Альтернатива** - использовать Docker-образ Astral:

```yaml
default:
  image: ghcr.io/astral-sh/uv:python3.12-bookworm-slim

before_script:
  - uv --version
```

Проще, чем ставить `uv` через `pip` в каждом этапе,
но создает привязку к образам Astral.

### Зачем нужен `--locked`

Флаг `--locked` заставляет uv проверить, что `uv.lock` соответствует
`pyproject.toml`. Если lockfile устарел (например, разработчик добавил
зависимость, но забыл обновить lockfile), команда завершится с ошибкой.

Это критически важно для CI:

- **без `--locked`**: uv молча пересоздаст lockfile и может установить другие
  версии зависимостей;
- **с `--locked`**: CI упадет, разработчик увидит проблему и обновит lockfile.

!!! warning "Всегда используйте `--locked` в CI"
    Без этого флага вы теряете гарантию воспроизводимости сборки. Это одна из
    самых частых ошибок при настройке CI с uv.

## Pre-commit и линтинг

Инструменты линтинга (`ruff`, `mypy`) удобно запускать
через `uvx` - это не требует их установки в проект.

### Запуск линтеров через `uvx`

`uvx` создает изолированное временное окружение для
инструмента и запускает его:

```bash
# Проверка стиля кода
uvx ruff check .

# Проверка форматирования
uvx ruff format --check .

# Автоформатирование
uvx ruff format .
```

Для `mypy` предпочтительнее использовать `uv run`,
так как ему нужен доступ к зависимостям проекта для
корректной проверки типов:

```bash
uv run mypy src/
```

### Интеграция с pre-commit

Для автоматической проверки кода при коммитах
используется `pre-commit`. Пример
`.pre-commit-config.yaml` с `ruff`:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.11.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

Установка и запуск через `uvx`:

```bash
# Установка хуков
uvx pre-commit install

# Ручной запуск на всех файлах
uvx pre-commit run --all-files
```

!!! tip "`uvx` вместо глобальной установки"
    `uvx pre-commit install` создает временное
    окружение для `pre-commit`. Хуки устанавливаются
    в `.git/hooks/` и работают автономно.

### Конфигурация `ruff` в `pyproject.toml`

Минимальная конфигурация `ruff` для uv-проекта:

```toml
[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]

[tool.ruff.format]
quote-style = "double"
```

### Конфигурация `mypy` в `pyproject.toml`

```toml
[tool.mypy]
python_version = "3.12"
strict = false
warn_unreachable = true
show_error_codes = true
ignore_missing_imports = true
```

## Таблица: Dockerfile-шаблоны

| Сценарий | Базовый образ | Установка uv | Зависимости | Особенности |
| -------- | ------------- | ------------ | ----------- | ----------- |
| Прототип | `uv:python3.12-bookworm-slim` | Встроен | `uv sync` | Минимальный Dockerfile |
| Single-stage | `python:3.12-slim` | `COPY --from` | `uv sync --frozen --no-dev` | Быстрая настройка |
| Multi-stage | `python:3.12-slim` | `COPY --from` (builder) | `uv sync --frozen` + cache-mount | Минимальный образ, нет uv в runtime |
| FastAPI | `python:3.12-slim` | `COPY --from` (builder) | `uv sync --frozen --no-dev` | Non-root user, `EXPOSE 8000` |
| C-расширения | `python:3.12-slim` | `COPY --from` (builder) | `apt install` + `uv sync` | `build-essential` в builder, `libpq5` в runtime |
| Security | `python:3.12-slim` | Installer (`curl`) | `uv sync --frozen` | Без сторонних base-образов |
