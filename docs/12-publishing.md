# Раздел 12. Сборка и публикация пакетов

## Сборка и публикация пакетов

### Подготовка пакета к сборке

Для сборки пакета необходимо наличие секции `[build-system]` в `pyproject.toml`.
Она указывает, какой build backend использовать:

=== "hatchling"

    ```toml
    [build-system]
    requires = ["hatchling"]
    build-backend = "hatchling.build"
    ```

=== "setuptools"

    ```toml
    [build-system]
    requires = ["setuptools>=75.0", "wheel"]
    build-backend = "setuptools.build_meta"
    ```

=== "flit"

    ```toml
    [build-system]
    requires = ["flit_core>=3.4"]
    build-backend = "flit_core.buildapi"
    ```

!!! note "Приложения vs библиотеки"
    Если вы создавали проект через `uv init`, секция `[build-system]`
    отсутствует - проект считается приложением. Для библиотек используйте `uv
    init --lib`, который добавит `[build-system]` автоматически.

### Метаданные пакета в pyproject.toml

Для публикации на PyPI пакет должен содержать корректные
метаданные в секции `[project]` файла `pyproject.toml`.
Обязательные и рекомендуемые поля:

```toml
[project]
name = "my-package"
version = "0.1.0"
description = "Краткое описание пакета"
authors = [
    { name = "Имя Автора", email = "author@example.com" },
]
license = { text = "MIT" }
requires-python = ">=3.10"
readme = "README.md"
classifiers = [
    "Programming Language :: Python :: 3",
    "License :: OSI Approved :: MIT License",
    "Operating System :: OS Independent",
]
```

| Поле | Обяз. | Описание |
| ---- | :-----: | -------- |
| `name` | да | Уникальное имя на PyPI |
| `version` | да | Версия (SemVer) |
| `description` | да | Однострочное описание |
| `authors` | рек. | Авторы пакета |
| `license` | рек. | Лицензия проекта |
| `requires-python` | рек. | Минимальная версия Python |
| `classifiers` | рек. | Классификаторы PyPI |
| `readme` | рек. | Путь к файлу README |

!!! tip "Управление версией"
    `uv version` позволяет обновить версию пакета
    перед публикацией:

    ```bash
    # Установить конкретную версию
    uv version 1.0.0

    # Увеличить минорную версию (1.2.3 -> 1.3.0)
    uv version --bump minor

    # Предпросмотр без изменения файла
    uv version 2.0.0 --dry-run
    ```

### Сборка пакета (`uv build`)

Команда `uv build` создает дистрибутивы в директории
`dist/`:

```bash
# Сборка sdist (.tar.gz) и wheel (.whl)
uv build

# Сборка только source distribution
uv build --sdist

# Сборка только wheel
uv build --wheel

# Сборка конкретного пакета в workspace
uv build --package my-package
```

Результат:

```text
dist/
  my_package-0.1.0.tar.gz            # sdist
  my_package-0.1.0-py3-none-any.whl  # wheel
```

**sdist** (source distribution) - архив с исходным кодом
и метаданными. Используется как fallback, когда wheel
не подходит для целевой платформы.

**wheel** - готовый к установке бинарный формат.
Устанавливается быстрее, так как не требует этапа сборки.

!!! warning "Очистка перед повторной сборкой"
    Перед сборкой новой версии удалите старые артефакты:

    ```bash
    rm -rf dist/
    uv build
    ```

    Иначе `uv publish` может загрузить устаревшие файлы.

!!! tip "Проверка сборки без `tool.uv.sources`"
    При публикации рекомендуется собирать с флагом
    `--no-sources`, чтобы убедиться, что пакет соберется
    через `pip` или другие инструменты:

    ```bash
    uv build --no-sources
    ```

### Публикация

Команда `uv publish` загружает собранные дистрибутивы в реестр пакетов:

```bash
# Публикация на PyPI (по умолчанию)
uv publish

# Публикация в приватный регистри
uv publish --index company-registry

# Публикация конкретных файлов
uv publish dist/mypackage-0.1.0-py3-none-any.whl
```

### Тестовая публикация на TestPyPI

Перед публикацией на PyPI рекомендуется проверить пакет
на TestPyPI - тестовом регистри с идентичным API.

**1. Настройка TestPyPI как индекса:**

```toml
# pyproject.toml
[[tool.uv.index]]
name = "testpypi"
url = "https://test.pypi.org/simple/"
publish-url = "https://test.pypi.org/legacy/"
explicit = true
```

**2. Публикация на TestPyPI:**

```bash
# Токен от https://test.pypi.org/manage/account/
export UV_PUBLISH_TOKEN="pypi-..."

uv publish --index testpypi
```

**3. Проверка установки:**

```bash
uv pip install \
    --index-url https://test.pypi.org/simple/ \
    my-package
python -c "import my_package"
```

**4. Публикация на production PyPI:**

```bash
export UV_PUBLISH_TOKEN="pypi-..."
uv publish
```

!!! note "Отдельные токены"
    TestPyPI и PyPI - разные сервисы с раздельными
    учетными записями. Токен от одного сервиса не подходит
    для другого.

### Аутентификация

Для публикации требуется аутентификация. uv поддерживает несколько способов:

```bash
# Через API-токен (рекомендуется для PyPI)
export UV_PUBLISH_TOKEN="pypi-AgEI..."

# Через логин/пароль (для приватных регистри)
export UV_PUBLISH_USERNAME="__token__"
export UV_PUBLISH_PASSWORD="pypi-AgEI..."
```

!!! tip "API-токены PyPI"
    PyPI рекомендует использовать API-токены вместо пароля. Создайте токен на
    странице [Account Settings](https://pypi.org/manage/account/) и используйте
    его через `UV_PUBLISH_TOKEN`.

### Trusted Publishing (GitHub Actions)

Trusted Publishing позволяет публиковать пакеты на PyPI
из GitHub Actions без API-токенов. Аутентификация происходит
через OpenID Connect (OIDC): GitHub подтверждает идентичность
workflow, а PyPI доверяет этому подтверждению.

**Настройка:**

1. На PyPI откройте настройки проекта: *Publishing* ->
   *Add a new publisher*.
2. Укажите владельца и имя репозитория, имя workflow-файла,
   имя environment (`pypi`).
3. В настройках GitHub-репозитория создайте environment
   `pypi`: *Settings* -> *Environments*.

**Пример workflow:**

```yaml
# .github/workflows/publish.yml
name: Publish

on:
  push:
    tags:
      - "v*"

jobs:
  publish:
    runs-on: ubuntu-latest
    environment:
      name: pypi
    permissions:
      id-token: write   # обязательно для OIDC
      contents: read
    steps:
      - uses: actions/checkout@v4

      - uses: astral-sh/setup-uv@v4

      - run: uv build

      # Smoke-тест собранного пакета
      - name: Проверка wheel
        run: >
          uv run --isolated --no-project
          --with dist/*.whl
          -- python -c "import my_package"

      # Токены не нужны - аутентификация через OIDC
      - run: uv publish
```

Для запуска публикации создайте тег:

```bash
git tag -a v0.1.0 -m "v0.1.0"
git push --tags
```

!!! tip "Преимущества Trusted Publishing"
    - Не нужно создавать и хранить API-токены.
    - Невозможна утечка токена через логи CI.
    - PyPI привязывает публикацию к конкретному
      репозиторию и workflow.

### Приватные регистри

uv поддерживает публикацию в приватные регистри
(Artifactory, GitLab Package Registry,
AWS CodeArtifact и др.).

**Настройка индекса с `publish-url`:**

```toml
# pyproject.toml
[[tool.uv.index]]
name = "company"
url = "https://registry.company.com/simple/"
publish-url = "https://registry.company.com/upload/"
```

**Публикация:**

```bash
uv publish --index company
```

**Аутентификация:**

```bash
# Через API-токен
export UV_PUBLISH_TOKEN="token-value"
uv publish --index company

# Через логин и пароль
export UV_PUBLISH_USERNAME="deploy"
export UV_PUBLISH_PASSWORD="secret"
uv publish --index company
```

| Переменная | Назначение |
| ---------- | ---------- |
| `UV_PUBLISH_TOKEN` | API-токен (приоритет над логином) |
| `UV_PUBLISH_USERNAME` | Имя пользователя |
| `UV_PUBLISH_PASSWORD` | Пароль |

**Интеграция с keyring:**

uv поддерживает получение учетных данных через `keyring`
при публикации:

```bash
uv publish --index company \
    --keyring-provider subprocess
```

Флаг `--keyring-provider subprocess` указывает uv
использовать CLI-утилиту `keyring` для получения
учетных данных. Альтернативно задается через переменную
`UV_KEYRING_PROVIDER`.

!!! note "Повторная публикация"
    Если загрузка прервалась, `uv publish` с `--check-url`
    пропустит уже загруженные файлы. При использовании
    `--index` URL индекса применяется как check URL
    автоматически.

### Полный цикл публикации

```bash
# 1. Обновите версию пакета
uv version --bump patch

# 2. Убедитесь, что тесты проходят
uv run pytest

# 3. Очистите старые артефакты и соберите пакет
rm -rf dist/
uv build

# 4. (опционально) Проверьте на TestPyPI
uv publish --index testpypi

# 5. Опубликуйте на PyPI
uv publish
```
