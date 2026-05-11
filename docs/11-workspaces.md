# Раздел 11. Workspaces

---

## Workspaces (monorepo)

### Что такое workspaces

Workspaces - это механизм для управления несколькими связанными пакетами
в одном репозитории (monorepo). Вместо того чтобы держать каждый пакет
в отдельном репозитории со своим lockfile и окружением, вы объединяете их
под общим корнем с единым `uv.lock`.

Типичные сценарии использования:

- микросервисная архитектура, где сервисы разделяют общие библиотеки;
- библиотека с несколькими подпакетами (core, utils, plugins);
- монорепозиторий компании с внутренними пакетами.

### Создание workspace с нуля

Пошаговый процесс на примере платформы из двух пакетов:
`my-core` (библиотека) и `my-cli` (CLI, использует core).

**Шаг 1.** Создать корневой каталог:

```bash
uv init --bare my-platform
cd my-platform
```

Флаг `--bare` создает минимальный `pyproject.toml` без
шаблонных файлов (`main.py` и т.п.).

**Шаг 2.** Превратить root в workspace - открыть
`pyproject.toml` и добавить нужные секции:

=== "pyproject.toml"

    ```toml
    [project]
    name = "my-platform-workspace"
    version = "0"
    requires-python = ">=3.12"

    [tool.uv]
    package = false

    [tool.uv.workspace]
    members = ["packages/*"]

    [dependency-groups]
    dev = [
        "pytest>=9.0",
        "ruff>=0.15",
    ]
    ```

=== "uv.toml"

    ```toml
    package = false

    [workspace]
    members = ["packages/*"]
    ```

**Шаг 3.** Создать пакеты-члены:

```bash
mkdir packages
uv init --lib packages/my-core
uv init --package packages/my-cli
```

`--lib` создает библиотеку с layout `src/my_core/`.
`--package` создает приложение, настроенное как устанавливаемый пакет.

**Шаг 4.** Связать пакеты - в `packages/my-cli/pyproject.toml` добавить зависимость:

=== "pyproject.toml"

    ```toml
    [project]
    name = "my-cli"
    version = "0.1.0"
    requires-python = ">=3.12"
    dependencies = [
        "my-core",
    ]

    [tool.uv.sources]
    my-core = { workspace = true }
    ```

=== "uv.toml"

    ```toml
    # Привязка workspace-пакетов указывается
    # в pyproject.toml через [tool.uv.sources]
    ```

Альтернативный способ - через CLI:

```bash
cd packages/my-cli
uv add my-core
```

`uv` автоматически обнаружит workspace member и добавит правильную запись в `[tool.uv.sources]`.

**Шаг 5.** Синхронизировать и проверить:

```bash
cd my-platform       # вернуться в root
uv sync
uv run --package my-cli python -c \
    "from my_core import hello; print(hello())"
```

### Структура workspace

```text
myworkspace/
  pyproject.toml          # корневой конфиг с workspace
  uv.lock                 # единый lockfile
  packages/
    lib-core/
      pyproject.toml      # конфиг пакета
      src/
        lib_core/
          __init__.py
    lib-utils/
      pyproject.toml
      src/
        lib_utils/
          __init__.py
    app-api/
      pyproject.toml
      src/
        app_api/
          __init__.py
```

### Конфигурация

В корневом `pyproject.toml` достаточно указать секцию `[tool.uv.workspace]`:

=== "pyproject.toml"

    ```toml
    [project]
    name = "myworkspace"
    version = "0.1.0"
    requires-python = ">=3.12"

    [tool.uv.workspace]
    members = ["packages/*"]
    ```

=== "uv.toml"

    ```toml
    [workspace]
    members = ["packages/*"]
    ```

Значение `members` принимает glob-паттерны. Каждый элемент, попавший под
паттерн, должен содержать собственный `pyproject.toml` с секцией `[project]`.

Можно указать несколько паттернов или исключить определенные пакеты:

=== "pyproject.toml"

    ```toml
    [tool.uv.workspace]
    members = ["packages/*", "services/*"]
    exclude = ["packages/legacy-*"]
    ```

=== "uv.toml"

    ```toml
    [workspace]
    members = ["packages/*", "services/*"]
    exclude = ["packages/legacy-*"]
    ```

### Единый lockfile

Одна из главных ценностей workspaces - **единый `uv.lock`** для всего
репозитория. Это гарантирует, что все пакеты используют согласованные версии
зависимостей, устраняя "dependency hell" при взаимодействии компонентов.

При выполнении `uv lock` в любом месте workspace `uv` найдет корневой
`pyproject.toml` и обновит общий lockfile.

### Зависимости между пакетами

Пакеты внутри workspace могут зависеть друг от друга.
Для этого используется секция `[tool.uv.sources]`:

=== "pyproject.toml"

    ```toml
    # packages/app-api/pyproject.toml
    [project]
    name = "app-api"
    version = "0.1.0"
    dependencies = [
        "lib-core",
        "lib-utils",
        "fastapi>=0.136",
    ]

    [tool.uv.sources]
    lib-core = { workspace = true }
    lib-utils = { workspace = true }
    ```

Директива `workspace = true` указывает uv, что пакет нужно брать из workspace,
а не из PyPI. При этом пакет устанавливается в режиме editable - изменения в
исходниках сразу видны зависимым пакетам.

### Editable-установка членов workspace

Все члены workspace устанавливаются как editable автоматически. Это значит:

- в `.venv/lib/python3.X/site-packages/` лежит не копия
  кода, а ссылка на исходники в `packages/<name>/src/`;
- изменения в файлах одного пакета сразу видны другим без
  переустановки;
- аналог `pip install -e .` для каждого пакета.

### Path-зависимости (без workspace)

Если пакетам нужны **разные** версии одной и той же зависимости
(например, один требует Django 4, другой - Django 5), workspace
не подходит. Альтернатива - path-зависимости:

=== "pyproject.toml"

    ```toml
    # packages/pkg-b/pyproject.toml
    [project]
    name = "pkg-b"
    dependencies = ["pkg-a"]

    [tool.uv.sources]
    pkg-a = { path = "../pkg-a", editable = true }
    ```

В этом случае каждый пакет имеет **собственный** `uv.lock` и **собственный** `.venv`.
Зависимости могут конфликтовать - каждый пакет резолвится независимо.

Минусы по сравнению с workspace:

- нет общего venv и lock - дольше синхронизировать;
- изменения в `pkg-a` требуют повторного `uv sync` в `pkg-b`;
- нет команды `uv run --package` из любого каталога.

### Циклические зависимости

Если `my-core` зависит от `my-utils`, а `my-utils` зависит от `my-core` -
`uv` выдаст ошибку при `uv lock`. Циклические зависимости между
пакетами запрещены. Если они возникают - границы между пакетами
проведены неправильно, либо это должно быть одним пакетом.

### Команды для работы с workspace

Все знакомые команды (`uv sync`, `uv run`, `uv add`, `uv lock`)
работают с workspace, но имеют дополнительные флаги.

**`uv sync`** - установка зависимостей:

```bash
# зависимости текущего пакета (по рабочему каталогу)
uv sync

# конкретный пакет (из любого каталога)
uv sync --package my-api

# все пакеты workspace
uv sync --all-packages
```

Когда какой вариант:

- `uv sync` - повседневная работа над текущим пакетом;
- `uv sync --package <name>` - в CI или Docker для сборки конкретного сервиса;
- `uv sync --all-packages` - полная инициализация монорепо (после первого clone).

**`uv run`** - запуск команд:

```bash
# из текущего каталога
uv run pytest

# из конкретного пакета workspace
uv run --package my-api pytest

# команду, определенную в одном из members
uv run my-cli
```

**`uv add` / `uv remove`** - управление зависимостями:

```bash
# добавить зависимость к текущему пакету (по cwd)
cd packages/api
uv add fastapi

# добавить к конкретному пакету из любого каталога
uv add --package my-api httpx

# добавить dev-зависимость в root
cd /path/to/workspace/root
uv add --dev pytest
```

**`uv lock`** - lockfile всегда один на весь workspace:

```bash
# обновить общий lock
uv lock

# проверить актуальность lock
uv lock --check
```

На каком бы пакете ни делался `uv add`, lockfile обновляется в root.

**`uv build`** - сборка для публикации:

```bash
# собрать конкретный пакет
uv build --package my-api

# проверка: сборка без workspace-источников
# (как будет у пользователя после pip install)
uv build --package my-api --no-sources
```

Если `--no-sources` падает - значит, workspace-зависимости не опубликованы
в PyPI, и внешние пользователи не смогут установить пакет.

### Фильтрация установки в workspace

Для Docker и CI полезны флаги, управляющие тем, **что** именно устанавливается:

```bash
# только зависимости, без самого проекта
uv sync --no-install-project

# без членов workspace (только внешние зависимости)
uv sync --no-install-workspace

# без конкретного пакета
uv sync --no-install-package my-internal-lib
```

Типичный сценарий - разделение Docker-слоев:

```dockerfile
# Слой 1: зависимости (меняется редко)
COPY pyproject.toml uv.lock ./
COPY packages/core/pyproject.toml ./packages/core/
COPY packages/api/pyproject.toml ./packages/api/
RUN uv sync --frozen --package my-api \
    --no-install-workspace --no-dev

# Слой 2: исходники (меняется часто)
COPY packages/core ./packages/core
COPY packages/api ./packages/api
RUN uv sync --frozen --package my-api --no-dev
```

Если вы находитесь в каталоге конкретного пакета, `uv` автоматически определит контекст:

```bash
cd packages/app-api
uv run pytest    # запуск в контексте app-api
```

### Типичные ошибки

**Общий `.venv` - побочный эффект workspace.**
Workspace всегда создает один общий `.venv` в корне. Все пакеты видят
зависимости друг друга. Python не обеспечивает изоляцию: пакет может
случайно импортировать зависимость, объявленную в другом member,
и это не вызовет ошибку локально. Проблема проявится только после
публикации или установки пакета отдельно.

**Конфликтующие зависимости.**
Workspace требует одного общего `uv.lock`. Если `my-api` хочет `pydantic>=2.5`,
а `my-cli` хочет `pydantic<2.0` - lockfile не соберется. Резолвер не найдет версию,
которая удовлетворит всех. Решение: привести требования к согласованности
или отказаться от workspace в пользу path-зависимостей.

**Имя пакета vs имя каталога.**
В `[tool.uv.sources]` нужно **имя пакета** из его `pyproject.toml`, а не имя каталога:

```toml
# pyproject.toml
[tool.uv.sources]
my-lib = { workspace = true }    # correct: имя пакета из [project].name
my_lib = { workspace = true }    # wrong: имя каталога, не совпадает с именем пакета
```

**Префикс `./` в members.**
Не используйте `./` в путях:

=== "pyproject.toml"

    ```toml
    [tool.uv.workspace]
    members = ["./packages/*"]    # wrong: uv не нормализует ./, пакеты не будут найдены
    members = ["packages/*"]      # correct: относительный путь без префикса
    ```

=== "uv.toml"

    ```toml
    [workspace]
    members = ["./packages/*"]    # wrong: uv не нормализует ./, пакеты не будут найдены
    members = ["packages/*"]      # correct: относительный путь без префикса
    ```

**`--locked` vs `--frozen` в Docker.**
В Docker-сборке workspace `--locked` может упасть с ошибкой
`references a workspace, but is not a workspace member`.
Причина: при копировании только `pyproject.toml` и `uv.lock`
(без каталогов members) `--locked` пытается провалидировать
workspace и не находит пакеты. Решение - `--frozen`:

```dockerfile
# --locked валидирует workspace и упадет, если members не скопированы
RUN uv sync --locked --package my-api
# --frozen пропускает валидацию, использует uv.lock как есть
RUN uv sync --frozen --package my-api
```

**Коллизии pytest.**
Если в нескольких members есть `tests/test_utils.py`, pytest падает
с `import file mismatch`. Решение - добавить в корневой `pyproject.toml`:

```toml
[tool.pytest.ini_options]
addopts = "--import-mode=importlib"
```

Или добавить `__init__.py` в каждый каталог `tests/`.

**`uv tool install` из workspace.**
`uv tool install ./packages/my-cli` создает ссылку на исходники. При удалении
workspace tool сломается. Правильный способ - сначала собрать, потом установить:

```bash
uv build --package my-cli
uv tool install ./dist/my_cli-0.1.0-py3-none-any.whl
```

## Когда использовать workspaces

### Критерии выбора

Workspace **подходит**, если:

- несколько пакетов в одном репо разделяют общий код;
- все пакеты совместимы по зависимостям (один `uv.lock`);
- нужна мгновенная видимость изменений между пакетами (editable install);
- команда хочет единую точку входа для сборки, тестов и линтинга.

Workspace **не подходит**, если:

- пакеты требуют конфликтующих версий зависимостей;
- нужна изоляция `.venv` между пакетами;
- пакеты концептуально не связаны (просто лежат в одном репо);
- разный темп релизов - один пакет обновляется ежедневно, другой раз в год.

### Сравнение подходов

| Подход | Lock | Venv | Конфликты deps | `--package` |
| ------ | ---- | ---- | -------------- | ----------- |
| Workspace | один общий | один общий | нельзя | да |
| Path-зависимости | по одному | по одному | можно | нет |
| Отдельные репо | по одному | по одному | можно | нет |

Для типичных монорепо с несколькими связанными пакетами (общий
стек, согласованные версии) - workspace правильный выбор. Для пакетов
с конфликтующими требованиями - path-зависимости без workspace.
