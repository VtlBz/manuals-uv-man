# Раздел 3. Инициализация проекта

---

## Команда `uv init`

Команда `uv init` создает новый Python-проект со всеми необходимыми файлами. Это
аналог `npm init` в мире **Node.js** или `cargo init` в **Rust**.

### Базовое использование

```bash
# Создание нового проекта в директории "myproject"
uv init myproject
```

Результат:

```text
Initialized project `myproject` at `/home/user/myproject`
```

Аргумент `myproject` определяет:

- имя директории, которая будет создана;
- значение `name` в `pyproject.toml`;
- prompt виртуального окружения (в `.venv/pyvenv.cfg`) -
  при активации терминал покажет `(myproject)`, а не `(.venv)`.

Также `uv init` автоматически выполняет `git init` и создает `.gitignore`
(с записью `.venv/`). Remote-репозиторий (origin) не добавляется - его
нужно указать вручную:

```bash
git remote add origin git@github.com:user/myproject.git
```

Чтобы отключить инициализацию git, используйте флаг `--no-vcs`.

Если вы уже находитесь в нужной директории, можно инициализировать проект на месте:

```bash
# Инициализация в текущей директории
mkdir myproject && cd myproject
uv init
```

Без аргумента `uv init` берет имя проекта из имени текущей директории.
В данном случае результат идентичен: `name = "myproject"`, `prompt = myproject`.

### Типы проектов: приложение vs библиотека

`uv` поддерживает два типа проектов, отличающихся структурой и назначением.

**1. Приложение** (`--app`):

```bash
# Создание проекта-приложения
uv init --app myapp
```

Приложение - это проект, который запускается, но не публикуется как пакет в PyPI.
Характерные черты:

- Плоская структура: `main.py` в корне проекта.
- Нет секции `[build-system]` в `pyproject.toml`.
- Типичные примеры: веб-сервисы, CLI-утилиты, скрипты автоматизации, data-пайплайны.

!!! note "hello.py в старых версиях"
    В версиях `uv` до 0.6.0 по умолчанию создавался файл `hello.py`.
    В актуальных версиях создается `main.py`.

Структура:

```text
myapp/
├── .python-version
├── README.md
├── main.py
└── pyproject.toml
```

**2. Библиотека** (`--lib`):

```bash
# Создание проекта-библиотеки
uv init --lib mylib
```

Библиотека - это проект, предназначенный для публикации и установки другими разработчиками.
Характерные черты:

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

Упакованное приложение - гибрид: проект, который запускается как программа,
но оформлен как устанавливаемый пакет.
Характерные черты:

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

После установки через `uv tool install .` команда `mycli` будет доступна в `PATH`.

### Сравнительная таблица шаблонов

| Флаг | Тип | build-system | project.scripts | Структура |
| ---- | --- | ------------ | --------------- | --------- |
| (без флага) | Приложение | Нет | Нет | Flat (`main.py` в корне) |
| `--app` | Приложение | Нет | Нет | Flat (`main.py` в корне) |
| `--lib` | Библиотека | Да (`uv_build`) | Нет | `src/` layout |
| `--package` | Пакет | Да (`uv_build`) | Да | `src/` layout |

!!! tip "Какой тип выбрать"
    Если вы пишете сервис, скрипт или приложение, которое будет запускаться
    напрямую, - выбирайте `--app`. Если вы создаете пакет, который другие
    разработчики будут устанавливать через `pip install` или `uv add`, -
    выбирайте `--lib`. По умолчанию (без флагов) `uv` создает приложение.

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
| `main.py` / `src/pkg/__init__.py` | Точка входа (зависит от типа проекта) |

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

Флаги можно комбинировать между собой и с `--app`, `--lib`, `--package`:

```bash
uv init --lib --no-readme --python 3.12 mylib
```

---

## Анатомия `pyproject.toml`

Файл `pyproject.toml` - центральный конфигурационный файл Python-проекта.
`uv` использует его как единственный источник правды о проекте,
его зависимостях и настройках.

### Секция `[project]`

Основные метаданные проекта, описанные стандартом [PEP 621](https://peps.python.org/pep-0621/):

```toml
[project]
name = "myapp"
version = "0.1.0"
description = "My awesome application"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "fastapi[standard]>=0.136",
    "pydantic>=2.0",
    "httpx>=0.28",
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

Группы зависимостей для разработки. Это относительно новый стандарт ([PEP 735](https://peps.python.org/pep-0735/)),
который `uv` поддерживает как основной способ организации dev-зависимостей:

```toml
[dependency-groups]
dev = [
    "pytest>=9.0",
    "ruff>=0.15",
    "mypy>=2.0",
]
test = [
    "pytest>=9.0",
    "pytest-cov>=7.1",
    "pytest-asyncio>=1.3",
]
lint = [
    "ruff>=0.15",
    "mypy>=2.0",
]
docs = [
    "mkdocs-material>=9.7",
]
```

По умолчанию `uv sync` устанавливает `[project.dependencies]` и группу `dev`.
Это поведение можно изменить через настройку `default-groups` в `[tool.uv]`.

Подробнее об управлении группами при установке (`uv sync`, `default-groups`,
`--no-dev`, `--group`, `--only-group`) - в разделе [Группы зависимостей](04-dependencies.md#группы-зависимостей).

### Секция `[project.optional-dependencies]`

Опциональные зависимости (extras) - предназначены для библиотек, позволяют
пользователям выбирать функциональность при установке:

```toml
[project.optional-dependencies]
ml = ["torch>=2.11", "transformers>=5.8"]
postgres = ["asyncpg>=0.31", "psycopg[binary]>=3.3"]
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

| Параметр | Назначение |
| -------- | ---------- |
| `resolution` | Стратегия выбора версий: `highest` (по умолчанию), `lowest`, `lowest-direct` |
| `override-dependencies` | Принудительная замена версий транзитивных зависимостей |
| `constraint-dependencies` | Дополнительные ограничения версий без установки |
| `default-groups` | Группы, устанавливаемые по умолчанию при `uv sync` |

Пример:

=== "pyproject.toml"

    ```toml
    [tool.uv]
    resolution = "highest"
    override-dependencies = ["urllib3>=2.0"]
    constraint-dependencies = ["numpy<2.0"]
    default-groups = ["dev", "test"]
    ```

=== "uv.toml"

    ```toml
    resolution = "highest"
    override-dependencies = ["urllib3>=2.0"]
    constraint-dependencies = ["numpy<2.0"]
    default-groups = ["dev", "test"]
    ```

Подробнее о каждом параметре - в разделе [Конфигурация](09-configuration.md).

### Секция `[build-system]`

Определяет, как собирать пакет для публикации:

```toml
[build-system]
requires = ["uv_build>=0.11.13,<0.12"]
build-backend = "uv_build"
```

- **Для библиотек** (`--lib`): обязательная секция. `uv` по умолчанию использует
  `uv_build`, но вы можете заменить его на `setuptools`, `hatchling`, `flit-core`,
  `pdm-backend` или любой другой PEP 517-совместимый бэкенд.
- **Для приложений** (`--app`): секция отсутствует, потому что приложения
  не публикуются как пакеты.

### Секции `[tool.ruff]` и `[tool.mypy]`

`pyproject.toml` - это общий конфигурационный файл для всей экосистемы Python. В
нем мирно сосуществуют настройки разных инструментов:

```toml
[project]
name = "myapp"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["fastapi[standard]>=0.136"]

[dependency-groups]
dev = ["pytest>=9.0", "ruff>=0.15", "mypy>=2.0"]

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
    `[tool.<name>]` и игнорирует остальные. `uv` работает точно так же - он читает
    секции `[project]`, `[dependency-groups]` и `[tool.uv]`, не трогая чужие настройки.

---

## Файл `uv.lock`

### Назначение

`uv.lock` - это lockfile, в котором зафиксированы точные версии **всех**
зависимостей проекта (прямых и транзитивных) для всех поддерживаемых платформ.

### Формат

Lockfile использует собственный формат `uv` (не `requirements.txt`). Файл
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
    В традиционном подходе приходилось вести отдельные файлы: `requirements.txt`,
    `requirements-dev.txt`, `constraints.txt`. С `pip-tools` вместо них - `requirements.in`
    и `requirements-dev.in` (`.txt` генерируется автоматически, но всё равно несколько
    файлов). `uv.lock` заменяет их все: он содержит информацию обо всех зависимостях,
    включая группы разработки, тестирования, линтинга - все в одном файле.

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
    Файл `uv.lock` **обязательно** должен быть в системе контроля версий.
    Это гарантирует, что:

    - Все члены команды работают с идентичным набором зависимостей.
    - CI/CD воспроизводит ту же среду, что и на машине разработчика.
    - Можно отследить, какие версии зависимостей менялись в каждом коммите.

    Не добавляйте `uv.lock` в `.gitignore`.

### Флаги `--locked` и `--frozen` для CI

В CI-окружениях важно контролировать, как `uv` работает с lockfile:

- `--locked` - проверяет актуальность `uv.lock` и устанавливает из него.
  Если lockfile устарел - ошибка.
- `--frozen` - устанавливает строго из `uv.lock` без проверки актуальности.

Подробно lock/sync workflow разбирается в разделе [Sync-workflow](06-sync-workflow.md).

---

## Структура проекта

### Приложение (`--app`)

```text
myapp/
├── .python-version   # Закрепленная версия Python (напр. "3.12")
├── .venv/            # Виртуальное окружение (создается при первом sync или run)
├── README.md         # Заготовка документации
├── main.py           # Точка входа
├── pyproject.toml    # Манифест проекта
└── uv.lock           # Lockfile (появляется после первого lock/sync/add)
```

Содержимое `main.py`:

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
        ├── __init__.py    # Инициализация пакета с публичным API
        └── py.typed       # Маркер PEP 561 для поддержки проверки типов
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

`uv` (и другие инструменты, например `pyenv`) читают этот файл, чтобы
определить, какую версию Python использовать в проекте. Автоматическое
скачивание недостающей версии зависит от настроек `python-preference` и
`python-downloads` (подробнее - в разделе [Управление версиями Python](08-python-versions.md)).

### Виртуальное окружение `.venv`

Каталог `.venv` не создается командой `uv init`. Он появляется при первом вызове:

```bash
# Любая из этих команд создаст .venv, если его нет
uv sync
uv run main.py
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
| `main.py` / `src/<pkg>/` | `uv init` | Да | Да |
| `README.md` | `uv init` | Да | Да |
| `.gitignore` | `uv init` | Да | Да |

---

## Интеграция uv в существующий проект

Если у вас уже есть проект с `pyproject.toml` (например, с настройками Ruff, mypy,
pytest), процедура интеграции `uv` описана в разделе [Миграция с существующим pyproject.toml](13-migration.md#работа-с-существующим-pyprojecttoml).

---
