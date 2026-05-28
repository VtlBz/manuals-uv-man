# Раздел 14. Диагностика и решение проблем

---

## Алгоритм диагностики

Пошаговая методика для случаев, когда что-то пошло
не так и непонятно, в какую сторону смотреть.

### Шаг 1. Проверить версию `uv`

```bash
uv --version
which uv
```

Часто проблема в устаревшей версии. Обновление (только для [standalone-установки](02-installation.md#автоматическое-обновление)):

```bash
uv self update
```

При установке через `pip`, Homebrew или системный менеджер
используйте соответствующий способ обновления.

### Шаг 2. Включить verbose-вывод

```bash
# INFO-уровень: решения резолвера, источники
uv sync -v

# DEBUG-уровень: внутренние операции, тайминги
uv sync -vv
```

Для сохранения вывода в файл:

```bash
uv sync -vv 2>&1 | tee uv-debug.log
```

Диагностические сообщения `uv` пишет в stderr, поэтому нужен `2>&1`.

### Шаг 3. Очистить кеш

```bash
uv cache clean
```

Закрывает все кейсы с поврежденными данными в кеше.

### Шаг 4. Пересоздать виртуальное окружение

```bash
rm -rf .venv
uv sync
```

Безопасно - окружение восстанавливается из `uv.lock`.
Подробнее об окружениях - в разделе [Виртуальные окружения](05-environments.md).

### Шаг 5. Проверить `pyproject.toml`

- Корректность `requires-python`.
- Отсутствие жестких пинов, вызывающих конфликты.
- Соответствие `.python-version` и `requires-python`.

### Шаг 6. Проверить `uv.lock`

Подробнее о флагах `--locked`, `--frozen`, `--check` - в разделе [Sync-workflow](07-sync-workflow.md).

```bash
# Актуальность lockfile
uv lock --check

# Пересоздать при необходимости
uv lock
```

Если lockfile поврежден или содержит конфликт после merge - удалить и пересоздать:

```bash
rm uv.lock
uv lock
```

### Шаг 7. Воспроизвести минимально

Если проблема не решена - создать изолированный воспроизводимый пример:

```bash
mkdir /tmp/repro && cd /tmp/repro
uv init test && cd test
uv add <проблемный-пакет>
```

Минимальный пример упрощает диагностику и помогает при создании Issue на GitHub.

---

## Типичные ошибки и решения

### ResolutionError / Conflicting dependencies

Типичное сообщение:

```text
× No solution found when resolving dependencies:
╰─▶ Because package-a==1.0 depends on lib>=2.0
    and package-b==0.5 depends on lib<2.0,
    we can conclude that requirements
    are unsatisfiable.
```

Чтение вывода - **снизу вверх**: от прямых
зависимостей проекта к корневой причине конфликта.

**Решения:**

```bash
# Кто тянет конфликтную зависимость
uv tree --invert --package <пакет>

# Обновить проблемный пакет
uv lock --upgrade-package <пакет>

# Подробный вывод резолвера
uv lock -vv
```

Частые причины:

- Жесткие пины (`==`) в `pyproject.toml`.
- Несовместимость `requires-python` с требованиями пакетов.
- Конфликт транзитивных зависимостей: слишком новый
  и слишком старый пакет в одном проекте.

Если конфликт нельзя разрешить обновлением пакетов или
ослаблением версионных ограничений, используйте override:

=== "pyproject.toml"

    ```toml
    [tool.uv]
    override-dependencies = [
        "problematic-package==1.5.0",
    ]
    ```

=== "uv.toml"

    ```toml
    override-dependencies = [
        "problematic-package==1.5.0",
    ]
    ```

!!! warning "Override-dependencies"
    Переопределение версий зависимостей - крайняя мера. Используйте только когда
    нет другого способа разрешить конфликт, и тщательно тестируйте результат.

### Python version not found / No Python in PATH

```text
error: No interpreter found for Python 3.13 in system path
```

**Решения:**

```bash
# Посмотреть установленные версии
uv python list --only-installed

# Установить нужную
uv python install 3.13

# Проверить, какую версию uv выбирает
uv python find
```

Также проверить:

- Не конфликтует ли `.python-version` с `requires-python`.
- Значение `python-preference` в конфигурации.
- Если используете pyenv - убедитесь, что нужная версия установлена
  и доступна через `pyenv global` или `pyenv local`.

### Resolved lockfile is not up to date

`uv.lock` не соответствует `pyproject.toml`. Возникает при
использовании `--locked` (обычно в CI).

**Решение:**

```bash
uv lock
git add uv.lock
git commit -m "Update uv.lock"
```

Эта ошибка означает, что кто-то изменил зависимости в `pyproject.toml`,
но не выполнил `uv lock`.

### Build backend not found

При попытке собрать пакет (`uv build`) или установить его в
editable-режиме возникает ошибка.

**Решение:** добавьте секцию `[build-system]` в `pyproject.toml`:

```toml
[build-system]
requires = ["uv_build>=0.11.14,<0.12"]
build-backend = "uv_build"
```

Эта секция необходима для библиотек. Приложения, созданные через `uv init`,
не имеют ее по умолчанию - это нормально, если вы не планируете собирать и
публиковать пакет.

### Package not found - приватный реестр

`uv` не находит пакет, который есть в корпоративном реестре.

**Решение:** проверьте конфигурацию индексов в `pyproject.toml`:

=== "pyproject.toml"

    ```toml
    [[tool.uv.index]]
    name = "company-registry"
    url = "https://pypi.company.com/simple/"
    ```

=== "uv.toml"

    ```toml
    [[index]]
    name = "company-registry"
    url = "https://pypi.company.com/simple/"
    ```

Убедитесь, что аутентификация настроена:

```bash
export UV_INDEX_COMPANY_REGISTRY_USERNAME="user"
export UV_INDEX_COMPANY_REGISTRY_PASSWORD="token"
```

### Ошибка парсинга .python-version (pyenv-virtualenv)

`uv` не может разобрать файл `.python-version`, потому что в нем
записано имя виртуального окружения pyenv (например, `myproject-3.12`).

В актуальных версиях `uv` (с 0.6+) такие записи игнорируются. Если используете
старую версию и получаете ошибку - обновите `uv`:

```bash
uv self update
```

Либо перезапишите `.python-version` чистой версией:

```bash
uv python pin 3.12
```

### Permission denied

Типичные причины:

- Попытка записи в системные каталоги.
- Неправильные права на каталог `.venv`.
- Ограничения файловой системы в контейнере.

**Решение:** убедиться, что `uv` работает с пользовательскими каталогами,
а `.venv` создан текущим пользователем.

### Сетевые ошибки (proxy, SSL)

#### certificate verify failed

Ошибка возникает при использовании корпоративных proxy с TLS-инспекцией
или self-signed сертификатов.

Решение: указать `uv` использовать системные сертификаты:

```bash
UV_SYSTEM_CERTS=true uv sync -vv
```

Или через конфигурацию:

=== "uv.toml"

    ```toml
    system-certs = true
    ```

=== "pyproject.toml"

    ```toml
    [tool.uv]
    system-certs = true
    ```

Для указания конкретного файла сертификатов:

```bash
SSL_CERT_FILE=/path/to/ca-bundle.crt uv sync
```

#### 401 / 403 при доступе к private index

Проверить credentials:

```bash
# Проверить, что переменные установлены
echo "$UV_INDEX_COMPANY_USERNAME"
echo "$UV_INDEX_COMPANY_PASSWORD"

# Запуск с подробным выводом
uv sync -vv
```

Убедитесь, что переменные окружения соответствуют имени индекса в конфигурации.
Для индекса с `name = "company"` переменные должны быть
`UV_INDEX_COMPANY_USERNAME` и `UV_INDEX_COMPANY_PASSWORD`.

#### Proxy

```bash
# HTTP/HTTPS proxy
HTTPS_PROXY=http://proxy.example.com:8080 uv sync

# Исключения из proxy
NO_PROXY=pypi.org,internal.example.com uv sync
```

#### No compatible wheel

Ошибка означает, что пакет не публикует wheel для текущей платформы.
`uv` попытается собрать из исходников, для чего могут потребоваться
build tools (`gcc`, `rust`, `cmake` и т.д.).

Типичные решения:

- Установить build-зависимости: `apt install build-essential`
- Для Rust-пакетов: `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`
- Проверить доступность wheels: `uv pip show --no-deps <package>`

**Проверка доступа к PyPI:**

```bash
curl -I https://pypi.org/simple/
```

### Повреждение `.venv`

Симптомы:

- `ModuleNotFoundError` при `uv run`, хотя пакет есть в lockfile.
- Битый симлинк на интерпретатор.
- Версия Python в `.venv` не совпадает с ожидаемой.

**Решение - пересоздать окружение:**

```bash
rm -rf .venv
uv sync
```

!!! tip "Безопасность пересоздания"
    Пересоздание `.venv` всегда безопасно - все
    зависимости восстанавливаются из `uv.lock`.
    Это первое, что стоит попробовать при любых
    проблемах с окружением.

### Медленный первый запуск

Первый `uv sync` или `uv add` работает заметно дольше последующих.

При первом запуске uv скачивает пакеты и заполняет локальный
кеш. Все последующие операции будут использовать кеш и работать значительно
быстрее. Это нормальное поведение.

### Merge-конфликты в uv.lock

Формат `uv.lock` (TOML с хешами) не поддается
ручному разрешению конфликтов. Надежная стратегия:

```bash
# 1. Принять любую версию lock
git checkout --theirs uv.lock

# 2. Разрешить конфликт в pyproject.toml (если есть)

# 3. Пересоздать lockfile
uv lock

# 4. Проверить, что окружение работает
uv sync
uv run pytest
```

!!! warning "Не редактируйте `uv.lock` вручную"
    Файл `uv.lock` генерируется автоматически. При merge-конфликтах не пытайтесь
    разрешить его вручную - примите любую из версий и выполните `uv lock`, чтобы
    пересчитать lockfile на основе актуального `pyproject.toml`.

### Hardlink в Docker

При использовании `--mount=type=cache` `uv` может выдать предупреждение
о невозможности hardlink между файловыми системами. Решение:

```dockerfile
ENV UV_LINK_MODE=copy
```

---

## Контекстные ошибки по разделам

Ниже - указатели на ошибки, описание которых находится в контексте
соответствующих разделов.

**Управление версиями Python ([Сосуществование с pyenv](03-python-versions.md#сосуществование-с-pyenv)):**

- Не работают стрелки/Backspace в REPL managed Python - проблема terminfo
  при прямом запуске без `uv run`.
- `pyenv: python3.12: command not found` при использовании shims - shim
  существует, но версия не установлена в pyenv.
- `uv` не парсит `.python-version` с именем виртуального окружения pyenv -
  плагин pyenv-virtualenv записывает имя env вместо версии.

**Sync-workflow ([Типичные сценарии](07-sync-workflow.md#типичные-сценарии)):**

- Разрешение конфликтов `uv.lock` после merge - см. также
  [Merge-конфликты в uv.lock](#merge-конфликты-в-uvlock) выше.

**Workspaces ([Типичные ошибки](13-workspaces.md#типичные-ошибки)):**

- Ошибки импорта из-за общего `.venv` в workspace - пакет случайно
  использует зависимость другого member.
- Конфликтующие зависимости в монорепо (один lockfile) - members требуют
  несовместимые версии одного пакета.
- Имя пакета не совпадает с каталогом в sources - в `[tool.uv.sources]`
  нужно имя из `[project].name`, а не имя каталога.
- `import file mismatch` в pytest - коллизия одноименных тестовых файлов
  в разных members.

---

## Управление кешем

Расположение кеша и настройка через переменные окружения -
в разделе [Управление кешем](09-configuration.md#управление-кешем).

Краткая справка по диагностическим командам:

| Ситуация | Команда |
| -------- | ------- |
| Кеш разросся | `uv cache prune` |
| Очистка в конце CI | `uv cache prune --ci` |
| Подозрение на коррупцию | `uv cache clean` |
| Пересобрать один пакет | `uv cache clean <pkg>` |
| Проверить свежие версии | `--refresh` |

Показать путь и размер кеша:

```bash
uv cache dir
du -sh "$(uv cache dir)"
```

---

## Отладка резолвера

### Verbose-вывод резолвера

```bash
# Подробный вывод процесса разрешения
uv lock -vv
```

В выводе обращать внимание на:

- Какие источники (PyPI, приватный индекс, git) проверяются.
- Где резолвер выполняет backtrack.
- Cache hits и misses.

### Дерево зависимостей

```bash
# Полное дерево
uv tree

# Обратное: кто тянет конкретный пакет
uv tree --invert --package <пакет>
```

`--invert` незаменим для поиска источника конфликта транзитивных зависимостей.

### Стратегии разрешения

Флаг `--resolution` управляет выбором версий:

```bash
# Минимальные совместимые версии
uv lock --resolution lowest

# Минимальные для прямых, максимальные
# для транзитивных
uv lock --resolution lowest-direct
```

`lowest` полезен для проверки нижних границ совместимости,
заявленных в `requires-python` и зависимостях.

### Воспроизводимость через `--exclude-newer`

```bash
# Только пакеты, опубликованные до указанной даты
uv lock --exclude-newer "2025-01-01"
```

Полезно для:

- Воспроизведения проблемы на конкретную дату.
- Исключения недавно опубликованных версий при подозрении на регрессию.

### Обновление отдельных пакетов

```bash
# Обновить один пакет в lockfile
uv lock --upgrade-package fastapi

# Обновить все зависимости
uv lock --upgrade
```

---

## Известные ограничения

### Зависимость от сети при первом запуске

После `uv cache clean` или на новой машине `uv` требует доступ к PyPI
для загрузки пакетов. Флаг `--frozen` запрещает обновление lockfile,
но не избавляет от необходимости скачивания.

### Кеш привязан к платформе

Кеш `uv` привязан к ОС и архитектуре. Wheels, скачанные на
macOS (arm64), не подходят для Linux (x86_64). В CI с разными
runner'ами кеш нужно разделять по ключу, включающему ОС.

### Аутентификация приватных индексов

Учетные данные передаются через переменные окружения с именем,
привязанным к имени индекса в `pyproject.toml`:

```bash
# Для индекса с name = "internal"
export UV_INDEX_INTERNAL_USERNAME=user
export UV_INDEX_INTERNAL_PASSWORD=token
```

В Docker credentials необходимо передавать через
`--mount=type=secret`, не через `--build-arg`.

---

## Аудит безопасности

Команда `uv audit` проверяет зависимости проекта на наличие известных уязвимостей.
По умолчанию используется база [OSV](https://osv.dev/) (Open Source Vulnerabilities).

```bash
# Аудит зависимостей текущего проекта
uv audit

# Аудит зафиксированных зависимостей (без пересоздания lockfile)
uv audit --locked

# Аудит без блокировки проекта
uv audit --frozen
```

Фильтрация и игнорирование уязвимостей:

```bash
# Игнорировать конкретную уязвимость по ID
uv audit --ignore GHSA-xxxx-yyyy-zzzz

# Игнорировать уязвимость, пока нет исправления
uv audit --ignore-until-fixed GHSA-xxxx-yyyy-zzzz

# Аудит только production-зависимостей (без dev)
uv audit --no-dev

# Аудит для конкретной платформы
uv audit --python-platform linux --python-version 3.12
```

Коды возврата:

- `0` - уязвимости не найдены.
- `1` - найдены уязвимости (полезно для CI - шаг упадет автоматически).

Использование в CI:

```yaml
- name: Аудит безопасности
  run: uv audit --locked
```

Если уязвимости найдены, `uv` выведет список затронутых пакетов
с идентификаторами CVE и рекомендациями по обновлению.

---
