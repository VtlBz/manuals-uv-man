# Раздел 8. Sync-workflow

---

## Команды sync-workflow

Sync-workflow в uv - это набор команд, обеспечивающих
воспроизводимую установку зависимостей. Три ключевые
команды покрывают весь жизненный цикл: от разрешения
зависимостей до экспорта для внешних инструментов.

| Команда | Назначение |
| ------- | ---------- |
| `uv lock` | Разрешить зависимости, записать lockfile |
| `uv sync` | Lock + установить пакеты в `.venv` |
| `uv export` | Экспорт lockfile в другие форматы |

### uv sync

`uv sync` без флагов выполняет полный цикл:

1. Читает `pyproject.toml`.
2. Проверяет актуальность `uv.lock`. Если lock устарел -
   пересчитывает и обновляет его.
3. Подбирает Python-интерпретатор (с учетом
   `.python-version`, `requires-python`).
4. Создает `.venv`, если его нет.
5. Устанавливает пакеты из `uv.lock` в `.venv`.
6. Удаляет из `.venv` лишние пакеты (exact sync).
7. Устанавливает сам проект как editable, если в
   `pyproject.toml` определена секция `[build-system]`.

Таким образом, `uv sync` - активная операция: она может
изменить и lockfile, и виртуальное окружение.

!!! note "Управление группами и extras"
    Команда `uv sync` поддерживает флаги `--no-dev`,
    `--group`, `--only-group`, `--all-groups`, `--extra`,
    `--all-extras` для выборочной установки зависимостей.
    Подробнее см.
    [раздел 5](05-dependencies.md).

### uv lock

`uv lock` выполняет только разрешение зависимостей и
запись `uv.lock`, без установки пакетов в окружение.
Подробнее - в разделе
раздел «uv lock в деталях» ниже.

### Экспорт lockfile

`uv export` конвертирует содержимое `uv.lock` в форматы,
понятные другим инструментам (например, `requirements.txt`).
Подробнее - в разделе [uv export](#uv-export) ниже.

---

## Флаги управления синхронизацией

### --frozen

```bash
uv sync --frozen
```

Поведение:

- **Не** проверяет актуальность `uv.lock` относительно
  `pyproject.toml`.
- **Не** пересчитывает зависимости.
- **Не** изменяет `uv.lock`.
- Устанавливает в `.venv` ровно то, что записано в
  lockfile.

Это самый предсказуемый режим: что зафиксировано в
lockfile, то и будет установлено. Подходит для ситуаций,
когда lockfile гарантированно корректен (прошел проверку
на предыдущем этапе или закоммичен в репозиторий).

!!! warning "Edge-case"
    Если `pyproject.toml` содержит `requests>=2.30`, а
    в `uv.lock` зафиксирована версия 2.20 (не
    удовлетворяющая диапазону), `uv sync --frozen` все
    равно установит 2.20. Для проверки соответствия
    используйте `--locked`.

### --locked

```bash
uv sync --locked
```

Поведение:

- Проверяет, что `uv.lock` актуален относительно
  `pyproject.toml`.
- Если lock актуален - устанавливает из него (аналогично
  `--frozen`).
- Если lock **не** актуален - завершается с ошибкой, ничего
  не устанавливает и не изменяет.

Пример сообщения об ошибке:

```text
error: The lockfile at `uv.lock` needs to be updated,
but `--locked` was provided. To update the lockfile,
run `uv lock`.
```

### Сравнение --frozen и --locked

| Сценарий | `--frozen` | `--locked` |
| -------- | ---------- | ---------- |
| Lock актуален | Устанавливает | Устанавливает |
| Lock устарел | Молча устанавливает | Ошибка, остановка |
| Lock отсутствует | Ошибка | Ошибка |
| Может изменить `uv.lock` | Нет | Нет |
| Дополнительная проверка | Нет | Да |

Мнемоника:

- `--frozen` - "lockfile заморожен, просто установи".
- `--locked` - "проверь, что lock консистентен, потом
  установи".

!!! tip "Зачем нужны оба флага"
    `--locked` выглядит универсальнее, но `--frozen`
    работает быстрее (пропускает проверку) и надежнее в
    edge-cases (Docker bind-mount, workspace-проекты).
    Распространенная стратегия: отдельная проверка через
    `uv lock --check` в начале CI, а `--frozen` на всех
    последующих этапах.

### --no-sync

Флаг `--no-sync` доступен у команд, которые по умолчанию
выполняют автоматическую синхронизацию (`uv run`,
`uv tool run`). Он пропускает этап установки:

```bash
# Запустить скрипт без предварительной синхронизации
uv run --no-sync pytest

# uv run без --no-sync автоматически делает inexact sync
```

Полезен, когда окружение заведомо актуально и нужно
сэкономить время запуска.

### Когда какой режим использовать

| Сценарий | Рекомендация | Причина |
| -------- | ------------ | ------- |
| Локальная разработка | `uv sync` (без флагов) | Авто-обновление lock и окружения |
| CI: проверка lock | `uv lock --check` | Быстрая проверка без установки |
| CI: установка для тестов | `uv sync --frozen` | Воспроизводимость после проверки |
| Docker | `uv sync --frozen` | Надежность, отсутствие edge-cases |
| Быстрый запуск скрипта | `uv run --no-sync` | Пропуск sync при актуальном `.venv` |

!!! note "Переменные окружения"
    Флаги `--frozen` и `--locked` можно задавать через
    переменные `UV_FROZEN=1` и `UV_LOCKED=1`. Подробнее
    см. [раздел 9](09-configuration.md).

### Exact и inexact синхронизация

По умолчанию `uv sync` выполняет **exact**-синхронизацию:
устанавливает все пакеты из lockfile и **удаляет** из
`.venv` все, чего в lockfile нет. Состояние `.venv` после
exact sync строго соответствует lockfile.

`uv run` по умолчанию выполняет **inexact**-синхронизацию:
устанавливает недостающее, но **не удаляет** лишние пакеты.
Это ускоряет каждый запуск.

```bash
# Запретить uv sync удалять лишние пакеты
uv sync --inexact
```

| Режим | Команда по умолчанию | Удаляет лишнее |
| ----- | -------------------- | -------------- |
| Exact | `uv sync` | Да |
| Inexact | `uv run` | Нет |

!!! tip "Практический эффект"
    Если пакет, установленный вручную через
    `uv pip install`, исчезает после `uv sync` - это
    ожидаемое поведение exact sync.

### --reinstall

Флаг `--reinstall` заново скачивает и устанавливает
пакеты, игнорируя кеш. Lockfile при этом не изменяется.

```bash
# Переустановить все пакеты
uv sync --reinstall

# Переустановить конкретный пакет
uv sync --reinstall-package fastapi
```

Сценарии использования:

- подозрение на повреждение кеша;
- обновление managed Python, когда требуется пересборка
  расширений;
- точечное исправление сломанного пакета без пересоздания
  всего окружения.

Альтернатива - полное пересоздание окружения:

```bash
rm -rf .venv && uv sync
```

### Флаги для Docker (многослойная установка)

Для оптимального кеширования Docker-слоев полезно
разделять установку зависимостей и самого проекта.

```bash
# Не устанавливать текущий проект (только зависимости)
uv sync --no-install-project

# В workspace: не устанавливать ни одного member
uv sync --no-install-workspace

# Не устанавливать конкретный пакет
uv sync --no-install-package my-lib
```

Типичный Dockerfile с двумя слоями:

```dockerfile
# Слой 1: зависимости (редко меняется, кешируется)
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-install-project --no-dev

# Слой 2: код проекта (меняется при каждом коммите)
COPY . .
RUN uv sync --frozen --no-dev
```

При изменении только исходного кода слой 1 берется из
кеша, устанавливается только сам проект.

---

## uv lock в деталях

### Что делает резолвер

При выполнении `uv lock` резолвер:

1. Анализирует все зависимости из `pyproject.toml`
   (прямые, групповые, optional).
2. Рекурсивно разрешает транзитивные зависимости.
3. Выбирает версии в соответствии со стратегией
   разрешения.
4. Записывает результат в `uv.lock`.

Если `uv.lock` уже существует, резолвер по умолчанию
сохраняет ранее зафиксированные версии - обновляет
только те зависимости, чьи constraints изменились.

### Формат uv.lock

Файл `uv.lock` использует формат TOML и имеет следующие
характеристики:

- **Universal** - содержит информацию обо всех платформах
  и версиях Python, поддерживаемых проектом.
- **Cross-platform** - один lockfile подходит для Linux,
  macOS и Windows.
- **Детерминированный** - одинаковый `pyproject.toml`
  всегда дает одинаковый `uv.lock`.
- **Человекочитаемый** - можно открыть и просмотреть
  зафиксированные версии.

Фрагмент `uv.lock`:

```toml
version = 1
requires-python = ">=3.12"

[[package]]
name = "requests"
version = "2.32.3"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "certifi" },
    { name = "charset-normalizer" },
    { name = "idna" },
    { name = "urllib3" },
]
```

!!! note "Файл `uv.lock` должен коммититься в репозиторий"
    В отличие от `.venv`, lockfile - часть исходного кода
    проекта. Он обеспечивает воспроизводимость сборки для
    всех участников команды.

### --upgrade и --upgrade-package

```bash
# Обновить все зависимости до последних допустимых версий
uv lock --upgrade

# Обновить конкретный пакет
uv lock --upgrade-package requests

# Обновить несколько пакетов
uv lock --upgrade-package requests \
        --upgrade-package httpx
```

!!! warning "Разница между `uv lock` и `uv lock --upgrade`"
    `uv lock` сохраняет текущие версии, если они все еще
    удовлетворяют constraints. `uv lock --upgrade`
    игнорирует текущие pinned-версии и выбирает
    максимально новые допустимые.

Подробнее об обновлении зависимостей см.
[раздел 5](05-dependencies.md).

### --resolution (стратегии разрешения)

Стратегия определяет, какие версии резолвер предпочитает.

| Стратегия | Поведение |
| --------- | --------- |
| `highest` | Максимально новые версии (умолчание) |
| `lowest` | Минимально допустимые версии |
| `lowest-direct` | Минимальные для прямых, максимальные для транзитивных |

```bash
# Разовый запуск с альтернативной стратегией
uv lock --resolution lowest

# Конфигурация в pyproject.toml
```

```toml
[tool.uv]
resolution = "lowest"
```

Стратегия `lowest` полезна для проверки совместимости с
нижними границами зависимостей. Подробнее см.
[раздел 5](05-dependencies.md).

### --check

```bash
uv lock --check
```

Проверяет актуальность `uv.lock` без изменения файлов
и без установки пакетов. Возвращает ненулевой код
возврата, если lockfile устарел или отсутствует.

Типичное применение:

- pre-commit hook;
- отдельная CI-джоба на ранней стадии пайплайна;
- скрипт проверки перед деплоем.

### Lock и sync в командной работе

Рекомендуемое распределение ответственности:

| Действие | Кто выполняет | Команда |
| -------- | ------------- | ------- |
| Обновление lockfile | Разработчик | `uv lock` / `uv add` |
| Коммит `uv.lock` | Разработчик | `git add uv.lock` |
| Проверка lock в CI | CI | `uv lock --check` |
| Установка из lock | CI / коллеги | `uv sync --frozen` |

Если разработчик изменил `pyproject.toml`, но забыл
обновить и закоммитить `uv.lock`, CI-проверка через
`uv lock --check` немедленно сигнализирует об ошибке.

---

## uv export

Команда `uv export` конвертирует содержимое `uv.lock` в
форматы, понятные инструментам, не поддерживающим
`uv.lock` напрямую.

### Экспорт в requirements.txt

```bash
# Базовый экспорт (формат requirements.txt по умолчанию)
uv export -o requirements.txt

# Явное указание формата
uv export --format requirements.txt -o requirements.txt
```

Результат содержит:

- точные версии всех пакетов (включая транзитивные);
- хеши для верификации (совместимость с
  `pip install --require-hashes`);
- маркеры окружения (`; python_version >= "3.12"`).

### Управление содержимым экспорта

```bash
# Без хешей
uv export --no-hashes -o requirements.txt

# Использовать существующий lock без проверки
uv export --frozen -o requirements.txt

# Без dev-зависимостей
uv export --no-dev -o requirements.txt
```

### Группы и extras при экспорте

```bash
# Включить дополнительную группу
uv export --group test -o requirements-test.txt

# Только конкретная группа (без production-зависимостей)
uv export --only-group test -o requirements-test.txt

# Исключить группу
uv export --no-group docs -o requirements.txt

# С extras
uv export --extra postgres -o requirements.txt
```

### Альтернативные форматы

```bash
# PEP 751 - стандартизированный формат lock-файлов
uv export --format pylock.toml -o pylock.toml

# CycloneDX SBOM - для security-аудита
uv export --format cyclonedx1.5 -o sbom.json
```

| Формат | Назначение |
| ------ | ---------- |
| `requirements.txt` | Совместимость с `pip` (умолчание) |
| `pylock.toml` | PEP 751, универсальный lockfile |
| `cyclonedx1.5` | SBOM для security/compliance |

### Сценарии использования

- **Совместимость** - передача зависимостей командам,
  использующим `pip` напрямую.
- **Security-сканирование** - многие сканеры (Snyk,
  Trivy, Dependabot) работают с `requirements.txt`
  или SBOM.
- **Docker без uv** - если в runtime-образе нет uv,
  зависимости устанавливаются через `pip`.
- **Передача артефактов** - в процессы или окружения,
  где uv недоступен.

!!! tip "Когда export не нужен"
    Если во всех окружениях (разработка, CI, Docker)
    доступен uv, используйте `uv sync --frozen` напрямую.
    Экспорт в `requirements.txt` добавляет лишний шаг
    и теряет кросс-платформенную информацию lockfile.

---

## Типичные сценарии

### Подключение нового разработчика

После клонирования репозитория достаточно одной команды:

```bash
git clone https://github.com/company/project.git
cd project
uv sync
```

`uv sync` автоматически:

- скачает нужную версию Python (если используется
  managed Python);
- создаст `.venv`;
- установит все зависимости из `uv.lock`.

### CI-пайплайн

Рекомендуемая последовательность шагов:

```yaml
steps:
  # 1. Проверка целостности lockfile
  - run: uv lock --check

  # 2. Установка зависимостей
  - run: uv sync --frozen --group test

  # 3. Запуск тестов
  - run: uv run pytest
```

Логика: `uv lock --check` ловит ситуацию "изменили
`pyproject.toml`, но забыли обновить lock". `--frozen`
на этапе установки гарантирует воспроизводимость без
повторной проверки.

### Docker (многослойная сборка)

```dockerfile
FROM python:3.12-slim

# Установка uv
COPY --from=ghcr.io/astral-sh/uv:latest \
     /uv /uvx /bin/

# Слой 1: зависимости (кешируется)
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-install-project --no-dev

# Слой 2: код проекта
COPY . .
RUN uv sync --frozen --no-dev

CMD ["uv", "run", "python", "-m", "myapp"]
```

Альтернативный подход - экспорт для образа без uv:

```dockerfile
FROM python:3.12-slim AS builder
COPY --from=ghcr.io/astral-sh/uv:latest \
     /uv /uvx /bin/
COPY pyproject.toml uv.lock ./
RUN uv export --frozen --no-dev --no-hashes \
    -o requirements.txt

FROM python:3.12-slim
COPY --from=builder requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "-m", "myapp"]
```

### Обновление одного пакета

```bash
# Обновить конкретный пакет в lockfile
uv lock --upgrade-package requests

# Применить обновление к окружению
uv sync

# Проверить установленную версию
uv run python -c \
    "import requests; print(requests.__version__)"
```

Если нужно обновить все зависимости:

```bash
uv lock --upgrade
uv sync
```

### Разрешение конфликтов после merge

Если после `git merge` или `git rebase` возникли
конфликты в `uv.lock`:

```bash
# Принять любую версию lock, затем пересоздать
git checkout --theirs uv.lock
uv lock

# Проверить, что окружение работает
uv sync
uv run pytest
```

!!! warning "Не редактируйте `uv.lock` вручную"
    Файл `uv.lock` генерируется автоматически. При
    merge-конфликтах не пытайтесь разрешить его вручную -
    примите любую из версий и выполните `uv lock`, чтобы
    пересчитать lockfile на основе актуального
    `pyproject.toml`.
