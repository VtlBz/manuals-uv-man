# Раздел 4. Инициализация проекта

---

## Команда `uv init`

Команда `uv init` создает новый Python-проект со всеми необходимыми файлами. Это
аналог `npm init` в мире Node.js или `cargo init` в Rust.

### Базовое использование

```bash
# Создание нового проекта в директории "myproject"
uv init myproject
```

Результат:

```text
Initialized project `myproject` at `/home/user/myproject`
```

Если вы уже находитесь в нужной директории,
можно инициализировать проект на месте:

```bash
# Инициализация в текущей директории
mkdir myproject && cd myproject
uv init
```

### Типы проектов: приложение vs библиотека

uv поддерживает два типа проектов, отличающихся структурой и назначением.

=== "Приложение (--app)"

    ```bash
    # Create an application project
    uv init --app myapp
    ```

    Приложение - это проект, который запускается, но не публикуется как
    пакет в PyPI. Характерные черты:

    - Плоская структура: `main.py` (или `hello.py`) в корне проекта.
    - Нет секции `[build-system]` в `pyproject.toml`.
    - Типичные примеры: веб-сервисы, CLI-утилиты, скрипты
      автоматизации, data-пайплайны.

    Структура:

    ```text
    myapp/
    ├── .python-version
    ├── README.md
    ├── hello.py
    └── pyproject.toml
    ```

=== "Библиотека (--lib)"

    ```bash
    # Create a library project
    uv init --lib mylib
    ```

    Библиотека - это проект, предназначенный для публикации и установки другими
    разработчиками. Характерные черты:

    - `src/`-layout: код лежит в `src/mylib/`.
    - Есть секция `[build-system]` с указанием бэкенда сборки.
    - Включает `py.typed` для поддержки type checking.
    - Типичные примеры: SDK, утилитарные библиотеки, фреймворки.

    Структура:

    ```text
    mylib/
    ├── .python-version
    ├── README.md
    ├── pyproject.toml
    └── src/
        └── mylib/
            ├── __init__.py
            └── py.typed
    ```

### Упакованное приложение (`--package`)

```bash
# Создание упакованного приложения
uv init --package mycli
```

Упакованное приложение - гибрид: проект, который
запускается как программа, но оформлен как
устанавливаемый пакет. Характерные черты:

- `src/`-layout, как у библиотеки.
- Есть секция `[build-system]` в `pyproject.toml`.
- Есть секция `[project.scripts]` с точкой входа
  (команда, доступная в `PATH` после установки).
- Типичные примеры: CLI-утилиты, распространяемые
  через `pip install` или `uv tool install`.

Структура:

```text
mycli/
├── .python-version
├── README.md
├── pyproject.toml
└── src/
    └── mycli/
        └── __init__.py
```

Пример секции `[project.scripts]` в `pyproject.toml`:

```toml
[project.scripts]
mycli = "mycli:main"
```

После установки через `uv tool install .` команда
`mycli` будет доступна в `PATH`.

### Сравнительная таблица шаблонов

| Флаг | Тип | build-system | project.scripts | Структура |
| ---- | --- | ------------ | --------------- | --------- |
| (без флага) | Приложение | Нет | Нет | Flat (`hello.py` в корне) |
| `--app` | Приложение | Нет | Нет | Flat (`hello.py` в корне) |
| `--lib` | Библиотека | Да (`hatchling`) | Нет | `src/` layout |
| `--package` | Пакет | Да (`hatchling`) | Да | `src/` layout |

!!! tip "Какой тип выбрать"
    Если вы пишете сервис, скрипт или приложение, которое будет запускаться
    напрямую, - выбирайте `--app`. Если вы создаете пакет, который другие
    разработчики будут устанавливать через `pip install` или `uv add`, -
    выбирайте `--lib`. По умолчанию (без флагов) uv создает приложение.

### Указание версии Python

Флаг `--python` позволяет задать версию Python при создании проекта:

```bash
# Создание проекта с фиксацией Python 3.12
uv init --app myapp --python 3.12
```

Эта команда:

- Запишет `3.12` в файл `.python-version`.
- Установит `requires-python = ">=3.12"` в `pyproject.toml`.
- Если Python 3.12 еще не установлен, uv скачает и установит его автоматически.

### Какие файлы генерирует `uv init`

| Файл | Назначение |
| ---- | ---------- |
| `pyproject.toml` | Манифест проекта: метаданные, зависимости, настройки инструментов |
| `.python-version` | Закрепленная версия Python для проекта |
| `README.md` | Заготовка документации |
| `hello.py` / `src/pkg/__init__.py` | Точка входа (зависит от типа проекта) |

!!! note "Что НЕ создает `uv init`"
    Команда `uv init` не создает `uv.lock` и `.venv`. Lockfile появится при
    первом вызове `uv lock`, `uv add` или `uv sync`. Виртуальное окружение
    `.venv` будет создано автоматически при первом `uv sync` или `uv run`.

### Дополнительные флаги

```bash
# Не создавать README.md
uv init --no-readme myproject

# Не инициализировать git-репозиторий
uv init --no-vcs myproject

# Указать описание проекта
uv init --description "Internal API client" myproject

# Подтянуть данные об авторе из git config
uv init --author-from auto myproject

# Инициализация в текущей директории
uv init .
```

Флаги можно комбинировать между собой и с `--app`,
`--lib`, `--package`:

```bash
uv init --lib --no-readme --python 3.12 mylib
```

---

## Анатомия `pyproject.toml`

Файл `pyproject.toml` - центральный конфигурационный файл Python-проекта. uv
использует его как единственный источник правды о
проекте, его зависимостях и настройках.

### Секция `[project]`

Основные метаданные проекта, описанные стандартом
[PEP 621](https://peps.python.org/pep-0621/):

```toml
[project]
name = "myapp"
version = "0.1.0"
description = "My awesome application"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "fastapi[standard]>=0.115",
    "pydantic>=2.0",
    "httpx>=0.27",
]
```

| Поле | Описание |
| ---- | -------- |
| `name` | Имя проекта (используется при публикации) |
| `version` | Текущая версия проекта |
| `description` | Краткое описание |
| `readme` | Путь к файлу README |
| `requires-python` | Минимальная поддерживаемая версия Python |
| `dependencies` | Список production-зависимостей |

### Секция `[dependency-groups]` (PEP 735)

Группы зависимостей для разработки. Это относительно новый стандарт
([PEP 735](https://peps.python.org/pep-0735/)), который uv поддерживает как
основной способ организации dev-зависимостей:

```toml
[dependency-groups]
dev = [
    "pytest>=8.0",
    "ruff>=0.8",
    "mypy>=1.13",
]
test = [
    "pytest>=8.0",
    "pytest-cov>=6.0",
    "pytest-asyncio>=0.24",
]
lint = [
    "ruff>=0.8",
    "mypy>=1.13",
]
docs = [
    "mkdocs-material>=9.5",
]
```

Группы можно использовать по отдельности или комбинировать:

```bash
# Синхронизация со всеми dev-зависимостями
uv sync

# Синхронизация с дополнительной группой test
uv sync --group test

# Синхронизация только группы lint
uv sync --only-group lint
```

### Секция `[project.optional-dependencies]`

Опциональные зависимости (extras) - предназначены для библиотек, позволяют
пользователям выбирать функциональность при установке:

```toml
[project.optional-dependencies]
ml = ["torch>=2.0", "transformers>=4.40"]
postgres = ["asyncpg>=0.29", "psycopg[binary]>=3.1"]
all = ["mylib[ml]", "mylib[postgres]"]
```

Потребители библиотеки устанавливают нужные extras:

```bash
# Установка библиотеки с ML extras
uv add "mylib[ml]"
```

!!! warning "Не путайте `dependency-groups` и `optional-dependencies`"
    - `[dependency-groups]` - для организации рабочего процесса разработчиков
      проекта (тестирование, линтинг, документация). Эти зависимости никогда не
      попадут к пользователям вашей библиотеки.
    - `[project.optional-dependencies]` - для конечных пользователей, которые
      устанавливают ваш пакет и хотят выбрать дополнительные возможности.

### Секция `[tool.uv]`

Настройки, специфичные для uv:

```toml
[tool.uv]
# Стратегия разрешения зависимостей
resolution = "highest"

# Принудительная установка версий для транзитивных зависимостей
override-dependencies = [
    "urllib3>=2.0",
]

# Добавление ограничений без установки
constraint-dependencies = [
    "numpy<2.0",
]

# Группы по умолчанию для `uv sync`
default-groups = ["dev", "test"]
```

Подробнее о конфигурации uv - в [разделе 9](09-configuration.md).

### Секция `[build-system]`

Определяет, как собирать пакет для публикации:

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

- **Для библиотек** (`--lib`): обязательная секция. uv по умолчанию использует
  `hatchling`, но вы можете заменить его на `setuptools`, `flit-core`,
  `pdm-backend` или любой другой PEP 517-совместимый бэкенд.
- **Для приложений** (`--app`): секция отсутствует, потому что
  приложения не публикуются как пакеты.

### Секции `[tool.ruff]` и `[tool.mypy]`

`pyproject.toml` - это общий конфигурационный файл для всей экосистемы Python. В
нем мирно сосуществуют настройки разных инструментов:

```toml
[project]
name = "myapp"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["fastapi[standard]>=0.115"]

[dependency-groups]
dev = ["pytest>=8.0", "ruff>=0.8", "mypy>=1.13"]

[tool.ruff]
line-length = 120
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP"]

[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
```

!!! tip "Один файл - все настройки"
    Не нужно держать отдельные `ruff.toml`, `mypy.ini`, `pytest.ini` - все
    конфигурации собраны в одном месте. Каждый инструмент читает свою секцию
    `[tool.<name>]` и игнорирует остальные. uv работает точно так же - он читает
    секции `[project]`, `[dependency-groups]` и
    `[tool.uv]`, не трогая чужие настройки.

---

## Файл `uv.lock`

### Назначение

`uv.lock` - это lockfile, в котором зафиксированы точные версии **всех**
зависимостей проекта (прямых и транзитивных) для всех поддерживаемых платформ.

### Формат

Lockfile использует собственный формат uv (не `requirements.txt`). Файл
человекочитаемый, но редактировать вручную его не нужно:

```toml
version = 1
requires-python = ">=3.12"

[[package]]
name = "anyio"
version = "4.7.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "idna" },
    { name = "sniffio" },
]
```

### Зачем нужен lockfile

В `pyproject.toml` зависимости указываются с диапазонами версий (`>=2.0,<3.0`).
Это гибко, но не обеспечивает воспроизводимость: сегодня резолвер выберет
`2.5.1`, а завтра - `2.6.0`, и что-нибудь может сломаться.

`uv.lock` фиксирует точные версии. Все разработчики в команде и CI-сервер
получат одинаковый набор пакетов - вне
зависимости от платформы и момента установки.

!!! note "Замена множеству requirements.txt"
    При работе с `pip-tools` приходилось вести отдельные файлы:
    `requirements.in` / `requirements.txt`, `requirements-dev.in` /
    `requirements-dev.txt` и т.д. `uv.lock` заменяет их все: он содержит
    информацию обо всех зависимостях, включая группы разработки,
    тестирования, линтинга - все в одном файле.

### Управление lockfile

```bash
# Создание или обновление lockfile
uv lock

# Обновление всех зависимостей до последних допустимых версий
uv lock --upgrade

# Обновление только конкретного пакета
uv lock --upgrade-package requests
```

`uv.lock` автоматически обновляется при:

- `uv add <package>` - добавление зависимости.
- `uv remove <package>` - удаление зависимости.
- `uv lock` - ручное обновление.

### Lockfile и Git

!!! warning "Коммитьте `uv.lock` в репозиторий"
    Файл `uv.lock` **обязательно** должен быть в системе
    контроля версий. Это гарантирует, что:

    - Все члены команды работают с идентичным набором зависимостей.
    - CI/CD воспроизводит ту же среду, что и на машине разработчика.
    - Можно отследить, какие версии зависимостей менялись в каждом коммите.

    Не добавляйте `uv.lock` в `.gitignore`.

### Флаги `--locked` и `--frozen` для CI

Для CI-окружений важно контролировать, как uv работает с lockfile:

```bash
# Ошибка, если lockfile не синхронизирован с pyproject.toml
uv sync --locked

# Не проверять актуальность lockfile, установить зафиксированное
uv sync --frozen
```

| Флаг | Поведение | Когда использовать |
| ---- | --------- | ------------------ |
| `--locked` | Проверяет, что `uv.lock` актуален. Если `pyproject.toml` изменился, а `uv.lock` не обновлен - завершается с ошибкой. | CI/CD: гарантия, что разработчик не забыл обновить lockfile |
| `--frozen` | Не проверяет актуальность lockfile. Устанавливает ровно то, что записано в `uv.lock`. | Docker-сборки, оффлайн-установки |

---

## Структура проекта

### Приложение (`--app`)

```text
myapp/
├── .python-version        # Pinned Python version (e.g., "3.12")
├── .venv/                 # Виртуальное окружение (auto-created on first sync)
├── README.md              # Project documentation placeholder
├── hello.py               # Entry point
├── pyproject.toml         # Project manifest
└── uv.lock                # Lockfile (after first lock/sync/add)
```

Содержимое `hello.py`:

```python
def main():
    print("Hello from myapp!")

if __name__ == "__main__":
    main()
```

### Библиотека (`--lib`)

```text
mylib/
├── .python-version
├── .venv/
├── README.md
├── pyproject.toml
├── uv.lock
└── src/
    └── mylib/
        ├── __init__.py    # Package init with version and public API
        └── py.typed       # PEP 561 marker for type checking support
```

Содержимое `src/mylib/__init__.py`:

```python
def hello() -> str:
    return "Hello from mylib!"
```

### Файл `.python-version`

Текстовый файл, содержащий одну строку с версией Python:

```text
3.12
```

uv (и другие инструменты, например `pyenv`) читают этот файл, чтобы определить,
какую версию Python использовать в проекте. При `uv run` или `uv sync` uv
автоматически скачает нужную версию, если она еще не установлена.

### Виртуальное окружение `.venv`

Каталог `.venv` не создается командой
`uv init`. Он появляется при первом вызове:

```bash
# Любая из этих команд создаст .venv, если его нет
uv sync
uv run hello.py
uv add requests
```

!!! tip "`.venv` и `.gitignore`"
    Директорию `.venv` следует добавить в `.gitignore`. Она содержит
    установленные пакеты и привязана к конкретной машине.

### Сводная таблица файлов проекта

| Файл | Кто создает | Редактировать | Коммитить в git |
| ---- | ----------- | ------------- | --------------- |
| `pyproject.toml` | `uv init` | Да | Да |
| `uv.lock` | `uv lock` / `uv add` / `uv sync` | Нет | Да (для приложений) |
| `.python-version` | `uv init` / `uv python pin` | Да | Да |
| `.venv/` | `uv sync` / `uv run` | Нет | Нет |
| `hello.py` / `src/<pkg>/` | `uv init` | Да | Да |
| `README.md` | `uv init` | Да | Да |
| `.gitignore` | `uv init` | Можно дополнять | Да |

---

## Интеграция uv в существующий проект

Частый сценарий: у команды уже есть проект с `pyproject.toml`, в котором
настроены Ruff и mypy, но зависимости управляются через `pip` +
`requirements.txt`. Как перейти на uv, не ломая существующие настройки?

!!! warning "`uv init` и существующий `pyproject.toml`"
    Если в директории уже есть `pyproject.toml`, команда
    `uv init` откажется выполняться, чтобы не перезаписать
    существующую конфигурацию. В этом случае добавляйте
    секции `[project]` и `[dependency-groups]`
    в `pyproject.toml` вручную, как показано ниже.

### Пример: до перехода

```toml
# pyproject.toml - existing config (only tool settings)
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
# pyproject.toml - after uv integration
[project]
name = "myapp"
version = "1.0.0"
description = "Our existing application"
requires-python = ">=3.12"
dependencies = [
    "fastapi[standard]>=0.115",
    "pydantic>=2.0",
    "sqlalchemy>=2.0",
    "httpx>=0.27",
    "celery>=5.4",
]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-cov>=6.0",
    "pytest-asyncio>=0.24",
    "ruff>=0.8",
    "mypy>=1.13",
    "pre-commit>=4.0",
]

# --- Existing tool configs below (untouched) ---

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

```bash
# Шаг 1: добавьте секции [project] и [dependency-groups] в pyproject.toml
# (отредактируйте вручную или пусть uv сделает это)

# Шаг 2: генерация lockfile
uv lock

# Шаг 3: создание окружения и установка всего
uv sync

# Шаг 4: проверка, что ruff и mypy работают
uv run ruff check .
uv run mypy src/
```

!!! note "Что происходит с настройками инструментов"
    Секции `[tool.ruff]`, `[tool.mypy]`, `[tool.pytest.ini_options]` остаются
    нетронутыми. uv работает только со своими секциями (`[project]`,
    `[dependency-groups]`, `[tool.uv]`, `[build-system]`) и
    не изменяет чужие конфигурации.

После успешной миграции старые файлы `requirements*.txt`
и `requirements*.in` можно удалить.
