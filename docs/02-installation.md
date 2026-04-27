# Раздел 2. Установка и настройка

---

## Системные требования

`uv` поставляется как нативный бинарный файл, скомпилированный на Rust.
Никаких дополнительных зависимостей (включая Python) для работы самого `uv` не требуется.

| Платформа | Архитектура | Минимальные требования |
| --- | --- | --- |
| Linux / WSL 2 | x86_64, aarch64 | glibc 2.17+ (CentOS 7+, Ubuntu 14.04+) |
| macOS | x86_64 (Intel), arm64 (Apple Silicon) | macOS 12 Monterey+ |
| Windows | x86_64 | Windows 10+ |

!!! note "Python не нужен для установки"
    В отличие от `pip`, `pipx` и большинства других менеджеров, `uv` не требует
    предустановленного Python. Более того, `uv` сам умеет скачивать и управлять
    версиями Python (об этом - в следующем разделе).

---

## Способы установки

### Standalone-установщик (рекомендуемый)

Это основной и рекомендуемый способ установки. Установщик скачивает
предкомпилированный бинарный файл для вашей
платформы и помещает его в `~/.local/bin/`.

=== "Linux / WSL 2"

    ```bash
    curl -LsSf https://astral.sh/uv/install.sh | sh
    ```

    Если `curl` недоступен, используйте `wget`:

    ```bash
    wget -qO- https://astral.sh/uv/install.sh | sh
    ```

    WSL 2 - это полноценная Linux-среда, установка идентична.

=== "macOS"

    ```bash
    curl -LsSf https://astral.sh/uv/install.sh | sh
    ```

=== "Windows"

    ```powershell
    powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
    ```

!!! tip "Проверка скрипта перед запуском"
    Если вы предпочитаете сначала просмотреть содержимое скрипта установки:

    ```bash
    curl -LsSf https://astral.sh/uv/install.sh | less
    ```

    Для Windows:

    ```powershell
    powershell -c "irm https://astral.sh/uv/install.ps1 | more"
    ```

Можно установить конкретную версию, указав её в URL
(здесь приведена актуальная версия на момент написания):

```bash
curl -LsSf https://astral.sh/uv/0.11.7/install.sh | sh
```

---

### Через pip / pipx

Установка через `pip` или `pipx` - это альтернативный вариант, удобный
если у вас уже настроена Python-среда.

```bash
# Установка через pipx (рекомендуется вместо pip)
pipx install uv
```

```bash
# Установка через pip
pip install uv
```

!!! warning "Ирония судьбы"
    Устанавливать `uv` через `pip` - это как вызывать эвакуатор, чтобы доехать
    до автосалона за новой машиной. Работает, но standalone-установщик проще и
    не зависит от наличия Python в системе.

---

### Через Homebrew (macOS)

```bash
brew install uv
```

!!! note "Версия в Homebrew"
    Версия `uv` в Homebrew может отставать от последнего релиза. Для получения
    самой свежей версии используйте standalone-установщик.

---

### Через системные менеджеры пакетов

=== "Linux (apt)"

    На момент написания `uv` отсутствует в стандартных репозиториях большинства
    дистрибутивов. Рекомендуется standalone-установщик.

=== "Linux (Arch / pacman)"

    ```bash
    # Доступен в community-репозитории
    pacman -S uv
    ```

=== "Windows (WinGet)"

    ```powershell
    winget install --id=astral-sh.uv -e
    ```

=== "Windows (Scoop)"

    ```powershell
    scoop install main/uv
    ```

!!! note "Особенности установки в WSL 2"
    WSL 2 - это полноценная Linux-среда, установка
    идентична Linux. Несколько практических замечаний:

    1. **Используйте Linux-установщик** - команда
       `curl ... | sh` работает как обычно.
    2. **Файловая система** - работайте в домашней
       директории Linux (`~/`), а не в примонтированных
       Windows-дисках (`/mnt/c/`). Производительность
       файловых операций на примонтированных дисках
       значительно ниже.
    3. **Не путайте окружения** - `uv`, установленный
       в WSL 2, и `uv`, установленный в Windows, работают
       независимо. Каждый видит только свои
       Python-интерпретаторы и виртуальные окружения.

---

## Проверка установки

После установки откройте новый терминал (или перезагрузите текущий) и выполните:

```bash
uv --version
```

Ожидаемый вывод (номер версии будет отличаться):

```text
uv 0.11.7 (4d4cd7037 2025-05-02)
```

Для просмотра списка всех доступных команд:

```bash
uv help
```

Вывод содержит краткое описание каждой подкоманды:

```text
An extremely fast Python package manager.

Usage: uv [OPTIONS] <COMMAND>

Commands:
  run      Run a command or script
  init     Create a new project
  add      Add dependencies to the project
  remove   Remove dependencies from the project
  sync     Update the project's environment
  lock     Update the project's lockfile
  ...
```

Для справки по конкретной команде используйте:

```bash
uv help run
# или
uv run --help
```

---

## Настройка PATH

### Куда устанавливается uv

Standalone-установщик размещает бинарные файлы
`uv` и `uvx` в следующих директориях:

| Платформа | Директория по умолчанию |
| --- | --- |
| Linux / macOS | `~/.local/bin/` |
| Windows | `%USERPROFILE%\.local\bin\` |

!!! note "Старые версии"
    До версии 0.5.0 `uv` устанавливался в `~/.cargo/bin/`. Если вы обновляете со
    старой версии, старые бинарники из `~/.cargo/bin/` не удаляются
    автоматически. Рекомендуется удалить их вручную, чтобы избежать конфликтов.

### Добавление в PATH

Установщик автоматически пытается добавить директорию в `PATH`. Если после
установки команда `uv` не найдена, добавьте путь вручную:

=== "bash"

    ```bash
    # Добавьте в ~/.bashrc
    echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
    source ~/.bashrc
    ```

=== "zsh"

    ```bash
    # Добавьте в ~/.zshrc
    echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
    source ~/.zshrc
    ```

=== "fish"

    ```fish
    # Добавьте в конфиг fish
    fish_add_path ~/.local/bin
    ```

=== "PowerShell"

    ```powershell
    # Добавьте в профиль PowerShell
    $BinPath = "$env:USERPROFILE\.local\bin"
    [Environment]::SetEnvironmentVariable("PATH", "$BinPath;$env:PATH", "User")
    ```

После изменения перезапустите терминал и проверьте:

```bash
which uv    # Linux / macOS
where uv    # Windows
```

---

## Автодополнение команд (Shell Completions)

`uv` поддерживает автодополнение команд и аргументов при нажатии `Tab`. Это
значительно ускоряет работу в терминале.

### Настройка для uv

=== "bash"

    Для текущей сессии:

    ```bash
    eval "$(uv generate-shell-completion bash)"
    ```

    Для постоянной активации:

    ```bash
    echo 'eval "$(uv generate-shell-completion bash)"' >> ~/.bashrc
    ```

=== "zsh"

    Для текущей сессии:

    ```bash
    eval "$(uv generate-shell-completion zsh)"
    ```

    Для постоянной активации:

    ```bash
    echo 'eval "$(uv generate-shell-completion zsh)"' >> ~/.zshrc
    ```

=== "fish"

    ```fish
    uv generate-shell-completion fish | source > ~/.config/fish/completions/uv.fish
    ```

=== "PowerShell"

    ```powershell
    if (!(Test-Path -Path $PROFILE)) {
      New-Item -ItemType File -Path $PROFILE -Force
    }
    Add-Content -Path $PROFILE -Value '(& uv generate-shell-completion powershell) | Out-String | Invoke-Expression'
    ```

### Настройка для uvx

`uvx` - это псевдоним для `uv tool run`.
Автодополнение для него настраивается отдельно:

=== "bash"

    ```bash
    echo 'eval "$(uvx --generate-shell-completion bash)"' >> ~/.bashrc
    ```

=== "zsh"

    ```bash
    echo 'eval "$(uvx --generate-shell-completion zsh)"' >> ~/.zshrc
    ```

=== "fish"

    ```fish
    uvx --generate-shell-completion fish | source > ~/.config/fish/completions/uvx.fish
    ```

После внесения изменений перезапустите терминал или перезагрузите конфигурацию:

```bash
source ~/.bashrc   # for bash
source ~/.zshrc    # for zsh
```

!!! tip "Проверка автодополнения"
    Введите `uv` и пробел, затем нажмите ++tab++ дважды. Вы должны увидеть
    список доступных подкоманд: `run`, `init`, `add`, `sync` и т.д.

---

## Обновление uv

### Автоматическое обновление

Если `uv` установлен через standalone-установщик, он умеет обновлять себя:

```bash
uv self update
```

Для установки конкретной версии (в том числе
откат на более старую):

```bash
uv self update 0.11.0
```

!!! warning "Только для standalone-установки"
    Команда `uv self update` работает только если `uv` был установлен через
    standalone-установщик. При установке через `pip`, `Homebrew` или системный
    менеджер пакетов используйте соответствующий способ обновления:

    ```bash
    # pip
    pip install --upgrade uv

    # Homebrew
    brew upgrade uv

    # Arch Linux
    pacman -Syu uv
    ```

!!! note "Переменная UV_NO_MODIFY_PATH"
    При обновлении `uv self update` повторно запускает установщик, который может
    модифицировать ваши shell-профили. Чтобы отключить это поведение:

    ```bash
    UV_NO_MODIFY_PATH=1 uv self update
    ```

---

## Удаление uv

Если вам потребуется удалить `uv`, выполните следующие шаги.

### Шаг 1: Очистка данных (опционально)

Перед удалением бинарных файлов рекомендуется очистить кеш и загруженные данные:

```bash
# Очистка кеша
uv cache clean

# Удаление скачанных версий Python
rm -r "$(uv python dir)"

# Удаление глобально установленных инструментов
rm -r "$(uv tool dir)"
```

### Шаг 2: Удаление бинарных файлов

=== "Linux / macOS"

    ```bash
    rm ~/.local/bin/uv ~/.local/bin/uvx
    ```

=== "Windows"

    ```powershell
    rm $HOME\.local\bin\uv.exe
    rm $HOME\.local\bin\uvx.exe
    rm $HOME\.local\bin\uvw.exe
    ```

=== "pip"

    ```bash
    pip uninstall uv
    ```

=== "Homebrew"

    ```bash
    brew uninstall uv
    ```

### Шаг 3: Очистка shell-профилей (опционально)

Если вы добавляли PATH и автодополнение вручную, удалите соответствующие строки
из `~/.bashrc`, `~/.zshrc` или аналогичного файла.

---
