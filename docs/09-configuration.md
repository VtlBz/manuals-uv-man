# Раздел 9. Конфигурация

---

## Иерархия конфигурации

uv поддерживает несколько уровней конфигурации. При наличии одной и той же
настройки на нескольких уровнях действует приоритет - более специфичный источник
перекрывает более общий.

### Порядок приоритетов (от высшего к низшему)

| Приоритет | Источник | Пример |
| --------- | -------- | ------ |
| 1 (высший) | Флаги командной строки | `uv sync --python 3.12` |
| 2 | Переменные окружения `UV_*` | `UV_PYTHON=3.12` |
| 3 | Проектная конфигурация (uv.toml) | `./uv.toml` или `./pyproject.toml` |
| 4 | Пользовательская конфигурация | `~/.config/uv/uv.toml` |
| 5 (низший) | Системная конфигурация | `/etc/uv/uv.toml` |

!!! warning "uv.toml vs pyproject.toml на уровне проекта"
    Если в директории проекта существуют **оба файла** - `uv.toml` и
    `pyproject.toml` с секцией `[tool.uv]`, то **`uv.toml` имеет приоритет**, а
    секция `[tool.uv]` в `pyproject.toml` **игнорируется**. Не используйте оба
    источника одновременно.

### Пути к конфигурационным файлам

=== "Linux / macOS"

    | Уровень | Путь |
    | ------- | ---- |
    | Проект | `<project>/uv.toml` или `<project>/pyproject.toml` |
    | Пользователь | `~/.config/uv/uv.toml` |
    | Система | `/etc/uv/uv.toml` |

=== "Windows"

    | Уровень | Путь |
    | ------- | ---- |
    | Проект | `<project>\uv.toml` или `<project>\pyproject.toml` |
    | Пользователь | `%APPDATA%\uv\uv.toml` |
    | Система | `%PROGRAMDATA%\uv\uv.toml` |

!!! note "Пользовательский и системный уровни"
    На пользовательском и системном уровнях допускается **только `uv.toml`**.
    Формат `pyproject.toml` с секцией `[tool.uv]` доступен только на уровне
    проекта.

---

## Конфигурация проекта в pyproject.toml

Для большинства проектов все настройки uv удобно хранить в `pyproject.toml`
рядом с конфигурацией других инструментов.

### Секция [tool.uv]

Основные настройки:

```toml
[tool.uv]
# Предпочтение версии Python
# managed      - предпочитать managed Python (по умолчанию)
# system       - предпочитать системный/pyenv Python
# only-managed - использовать ТОЛЬКО managed Python
# only-system  - использовать ТОЛЬКО системный Python
python-preference = "managed"

# Автоматическое скачивание Python
# automatic - скачивать при необходимости (по умолчанию)
# manual    - скачивать только по явному запросу
# never     - никогда не скачивать, ошибка если не найден
python-downloads = "automatic"
```

### Индексы пакетов [[tool.uv.index]]

Для подключения дополнительных или приватных индексов:

```toml
[[tool.uv.index]]
name = "company-registry"
url = "https://pypi.company.com/simple/"
```

Подробнее об индексах - в разделе «Приватные индексы пакетов» ниже.

### Источники зависимостей [tool.uv.sources]

Позволяет указать альтернативные источники для конкретных пакетов:

```toml
[tool.uv.sources]
# Git-репозиторий
my-lib = { git = "https://github.com/company/my-lib.git", tag = "v1.2.0" }

# Локальный путь (editable-установка)
shared-utils = { path = "../shared-utils", editable = true }

# Прямой URL
special-package = { url = "https://files.company.com/special-package-1.0.tar.gz" }
```

!!! note "Sources vs dependencies"
    Секция `[tool.uv.sources]` не заменяет `[project.dependencies]`. Пакет
    должен быть указан в обоих местах: в `dependencies` - как требование, в
    `sources` - как альтернативный источник. При публикации пакета `sources`
    игнорируется, и используются стандартные индексы.

### Секция [tool.uv.pip]

Настройки, специфичные для pip-совместимого интерфейса (`uv pip install`, `uv
pip compile` и т.д.):

```toml
[tool.uv.pip]
# URL индекса для pip-интерфейсных команд
index-url = "https://pypi.company.com/simple/"
extra-index-url = ["https://pypi.org/simple/"]

# Другие настройки pip-интерфейса
no-build-isolation = false
```

!!! warning "Разделение настроек"
    Секция `[tool.uv.pip]` влияет **только** на команды `uv pip *`. Команды
    проектного интерфейса (`uv add`, `uv sync`, `uv lock`) используют
    `[[tool.uv.index]]` для настройки индексов. Это два отдельных контекста
    конфигурации.

---

## uv.toml vs pyproject.toml

uv поддерживает два формата конфигурации на уровне проекта: `uv.toml` и секцию
`[tool.uv]` в `pyproject.toml`. Настройки идентичны, отличается только
синтаксис.

### Сравнение синтаксиса

=== "pyproject.toml"

    ```toml
    [tool.uv]
    python-preference = "managed"
    python-downloads = "automatic"

    [[tool.uv.index]]
    name = "company"
    url = "https://pypi.company.com/simple/"

    [tool.uv.pip]
    index-url = "https://pypi.company.com/simple/"
    ```

=== "uv.toml"

    ```toml
    python-preference = "managed"
    python-downloads = "automatic"

    [[index]]
    name = "company"
    url = "https://pypi.company.com/simple/"

    [pip]
    index-url = "https://pypi.company.com/simple/"
    ```

Ключевое отличие: в `uv.toml` отсутствует префикс `[tool.uv]` - настройки
размещаются на верхнем уровне.

### Когда использовать какой формат

| Сценарий | Рекомендация |
| -------- | ------------ |
| Все настройки в одном месте | `pyproject.toml` с секцией `[tool.uv]` |
| Выделенная конфигурация только для uv | `uv.toml` |
| Пользовательский уровень (`~/.config/uv/`) | Только `uv.toml` |
| Системный уровень (`/etc/uv/`) | Только `uv.toml` |
| В проекте уже есть `pyproject.toml` | `pyproject.toml` (удобнее - все в одном файле) |
| Проект без `pyproject.toml` (standalone-скрипты) | `uv.toml` |

!!! warning "Не смешивайте форматы"
    Не создавайте `uv.toml` и `[tool.uv]` в `pyproject.toml` одновременно. Если
    `uv.toml` существует, секция `[tool.uv]` в `pyproject.toml` полностью
    игнорируется. Это может привести к путанице, когда вы меняете настройки в
    `pyproject.toml`, а они не применяются.

---

## Переменные окружения

uv поддерживает переменные окружения с префиксом `UV_` для всех основных
настроек. Они удобны для CI/CD, Docker и временного изменения поведения.

### Управление Python

| Переменная | Описание | Пример |
| ---------- | -------- | ------ |
| `UV_PYTHON` | Путь к интерпретатору Python | `UV_PYTHON=/usr/bin/python3.12` |
| `UV_PYTHON_PREFERENCE` | Предпочтение версий Python | `UV_PYTHON_PREFERENCE=system` |
| `UV_PYTHON_DOWNLOADS` | Автоматическая загрузка Python | `UV_PYTHON_DOWNLOADS=never` |

### Окружение и проект

| Переменная | Описание | Пример |
| ---------- | -------- | ------ |
| `UV_PROJECT_ENVIRONMENT` | Путь к виртуальному окружению | `UV_PROJECT_ENVIRONMENT=/opt/venv` |
| `UV_FROZEN` | Эквивалент `--frozen` | `UV_FROZEN=1` |
| `UV_LOCKED` | Эквивалент `--locked` | `UV_LOCKED=1` |

### Кеш

| Переменная | Описание | Пример |
| ---------- | -------- | ------ |
| `UV_CACHE_DIR` | Путь к директории кеша | `UV_CACHE_DIR=/tmp/uv-cache` |
| `UV_NO_CACHE` | Отключение кеша | `UV_NO_CACHE=1` |
| `UV_LINK_MODE` | Режим линковки из кеша | `UV_LINK_MODE=copy` |

### Индексы и аутентификация

| Переменная | Описание | Пример |
| ---------- | -------- | ------ |
| `UV_INDEX_URL` | URL индекса по умолчанию | `UV_INDEX_URL=https://pypi.company.com/simple/` |
| `UV_EXTRA_INDEX_URL` | Дополнительные индексы | `UV_EXTRA_INDEX_URL=https://pypi.org/simple/` |
| `UV_INDEX_{NAME}_USERNAME` | Имя пользователя для именованного индекса | `UV_INDEX_COMPANY_USERNAME=deploy` |
| `UV_INDEX_{NAME}_PASSWORD` | Пароль для именованного индекса | `UV_INDEX_COMPANY_PASSWORD=secret` |

### Файлы .env

| Переменная | Описание | Пример |
| ---------- | -------- | ------ |
| `UV_ENV_FILE` | Путь к `.env`-файлу | `UV_ENV_FILE=.env.production` |
| `UV_NO_ENV_FILE` | Отключить загрузку `.env` | `UV_NO_ENV_FILE=1` |

### Примеры использования

=== "CI/CD (GitHub Actions)"

    ```yaml
    env:
      UV_FROZEN: "1"              # Don't update lockfile
      UV_CACHE_DIR: /tmp/uv-cache # Cacheable path
      UV_PYTHON: "3.12"           # Pin Python version

    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv sync
      - run: uv run pytest
    ```

=== "Docker"

    ```dockerfile
    FROM python:3.12-slim

    ENV UV_FROZEN=1
    ENV UV_NO_CACHE=1
    ENV UV_LINK_MODE=copy
    ENV UV_PYTHON_DOWNLOADS=never

    COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
    COPY pyproject.toml uv.lock ./
    RUN uv sync --no-dev --no-editable
    ```

=== "Локальная разработка"

    ```bash
    # Временное использование системного Python
    UV_PYTHON_PREFERENCE=system uv sync

    # Временное отключение кеша
    UV_NO_CACHE=1 uv add flask
    ```

---

## Приватные индексы пакетов

Если ваша команда использует внутренний реестр пакетов, uv поддерживает его
подключение с гибкой настройкой приоритетов и аутентификации.

### Добавление индекса

Базовая конфигурация в `pyproject.toml`:

```toml
[[tool.uv.index]]
name = "company-registry"
url = "https://pypi.company.com/simple/"
```

По умолчанию этот индекс будет использоваться **в дополнение** к PyPI. uv будет
искать пакеты сначала в добавленных индексах (в порядке объявления), затем в
PyPI.

### Флаг default - замена PyPI

Если вы хотите, чтобы ваш индекс использовался **вместо** PyPI по умолчанию:

```toml
[[tool.uv.index]]
name = "company"
url = "https://pypi.company.com/simple/"
default = true
```

С `default = true` этот индекс заменяет PyPI. Если пакет не найден в нем, uv
**не будет** искать его на PyPI (если только PyPI не добавлен отдельно как
дополнительный индекс).

### Флаг explicit - индекс только для определенных пакетов

Для специализированных индексов (например, PyTorch с CUDA-сборками) используйте
`explicit`:

```toml
[[tool.uv.index]]
name = "pytorch"
url = "https://download.pytorch.org/whl/cu118"
explicit = true
```

Индекс с `explicit = true` используется **только** для пакетов, которые явно
ссылаются на него через `[tool.uv.sources]`:

```toml
[tool.uv.sources]
torch = { index = "pytorch" }
torchvision = { index = "pytorch" }
```

Это предотвращает случайную установку пакетов из неожиданных индексов.

### Пример комплексной конфигурации

```toml
[project]
name = "ml-service"
version = "1.0.0"
requires-python = ">=3.12"
dependencies = [
    "flask>=3.1",
    "torch>=2.5",
    "company-ml-utils>=0.3",
]

# Внутренний регистри с корпоративными пакетами
[[tool.uv.index]]
name = "company"
url = "https://pypi.company.com/simple/"

# Индекс PyTorch с CUDA-сборками (explicit - только для torch/torchvision)
[[tool.uv.index]]
name = "pytorch"
url = "https://download.pytorch.org/whl/cu118"
explicit = true

[tool.uv.sources]
torch = { index = "pytorch" }
torchvision = { index = "pytorch" }
```

### Аутентификация

Для приватных индексов, требующих аутентификации, доступны несколько способов.

#### Способ 1. Переменные окружения (рекомендуемый)

Имя переменной формируется из имени индекса: `UV_INDEX_{NAME}_USERNAME` и
`UV_INDEX_{NAME}_PASSWORD`. Имя приводится к верхнему регистру, дефисы
заменяются на подчеркивания.

```bash
# Для индекса с именем "company-registry"
export UV_INDEX_COMPANY_REGISTRY_USERNAME="deploy-user"
export UV_INDEX_COMPANY_REGISTRY_PASSWORD="deploy-token"
```

В CI/CD эти значения хранятся в секретах:

```yaml
# GitHub Actions
env:
  UV_INDEX_COMPANY_REGISTRY_USERNAME: ${{ secrets.PYPI_USERNAME }}
  UV_INDEX_COMPANY_REGISTRY_PASSWORD: ${{ secrets.PYPI_TOKEN }}
```

#### Способ 2. Файл .netrc

Создайте файл `~/.netrc` с учетными данными:

```text
machine pypi.company.com
    login deploy-user
    password deploy-token
```

```bash
# Убедитесь в правильности прав доступа
chmod 600 ~/.netrc
```

uv автоматически читает `.netrc` при обращении к хостам, указанным в файле.

#### Способ 3. Интеграция с keyring

uv поддерживает Python-пакет `keyring` для безопасного хранения учетных данных:

```bash
# Установка keyring
uv tool install keyring

# Сохранение учетных данных
keyring set https://pypi.company.com/simple/ deploy-user
```

#### Способ 4. Учетные данные в URL (не рекомендуется)

```toml
[[tool.uv.index]]
name = "company"
url = "https://user:password@pypi.company.com/simple/"
```

!!! warning "Небезопасно"
    Встраивание учетных данных в URL **крайне не рекомендуется**, так как они
    попадут в `pyproject.toml` и могут быть закоммичены в репозиторий.
    Используйте этот способ только для локального тестирования, если другие
    варианты недоступны.

### Индексы для pip-интерфейса

Настройки индексов в `[[tool.uv.index]]` используются только для проектных
команд (`uv add`, `uv sync`, `uv lock`). Для pip-совместимого интерфейса (`uv
pip install`, `uv pip compile`) настройте индексы отдельно:

```toml
[tool.uv.pip]
index-url = "https://pypi.company.com/simple/"
extra-index-url = ["https://pypi.org/simple/"]
```

---

## Управление кешем

uv использует глобальный кеш для хранения загруженных пакетов, скомпилированных
wheel-файлов и метаданных. Это значительно ускоряет повторные операции.

### Расположение кеша

```bash
# Показать путь к директории кеша
uv cache dir
```

Пути по умолчанию:

| Платформа | Путь |
| --------- | ---- |
| Linux | `~/.cache/uv/` |
| macOS | `~/Library/Caches/uv/` |
| Windows | `%LOCALAPPDATA%\uv\cache\` |

Для изменения расположения используйте переменную окружения:

```bash
export UV_CACHE_DIR="/opt/uv-cache"
```

### Как работает кеш

uv хранит пакеты в кеше **глобально** - один экземпляр пакета на всю систему. В
виртуальные окружения проектов создаются ссылки (hardlink или symlink) на файлы
из кеша. Это обеспечивает:

- экономию дискового пространства - пакет хранится один раз, даже если
  используется в 10 проектах;
- скорость - установка из кеша занимает миллисекунды.

### Режим линковки (UV_LINK_MODE)

| Режим | Описание | Когда использовать |
| ----- | -------- | ------------------ |
| `hardlink` | Жесткие ссылки (по умолчанию) | Стандартный режим, самый быстрый |
| `symlink` | Символические ссылки | Альтернатива hardlink |
| `copy` | Копирование файлов | Когда кеш и проект на разных файловых системах (Docker, NFS) |

```bash
# Задать режим ссылок глобально
export UV_LINK_MODE=copy

# Или в pyproject.toml
```

```toml
[tool.uv]
link-mode = "copy"
```

!!! tip "Docker и UV_LINK_MODE"
    В многослойных Docker-сборках часто кеш и файловая система проекта находятся
    на разных слоях. В таких случаях hardlink не работает - используйте
    `UV_LINK_MODE=copy` или `--link-mode copy`.

### Команды управления кешем

```bash
# Показать расположение и структуру кеша
uv cache dir

# Очистка всего кеша
uv cache clean

# Удаление только устаревших записей (безопасно запускать периодически)
uv cache prune

# Очистка кеша для конкретного пакета
uv cache clean flask
```

### Проверка размера кеша

```bash
# Показать директорию кеша
uv cache dir

# Проверка размера кеша
du -sh "$(uv cache dir)"
```

!!! tip "Периодическая очистка"
    Команда `uv cache prune` безопасна для регулярного использования - она
    удаляет только записи, которые не используются ни одним из проектов. Полная
    очистка (`uv cache clean`) потребует повторной загрузки пакетов при
    следующем `uv sync`.

---

## Отключение конфигурации (--no-config)

Флаг `--no-config` полностью отключает чтение конфигурационных файлов всех
уровней. uv будет использовать только значения по умолчанию, переменные
окружения и флаги командной строки.

```bash
# Игнорировать все файлы конфигурации
uv sync --no-config

# Полезно для отладки проблем с конфигурацией
uv lock --no-config --verbose
```

Это полезно для:

- диагностики проблем с конфигурацией - если команда работает с `--no-config`,
  но не без него, проблема в конфигурационном файле;
- воспроизведения поведения на "чистой" системе;
- CI/CD, где вы хотите полностью контролировать настройки через переменные
  окружения.
