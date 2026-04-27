# Курсовые материалы: 11-workspaces.md

!!! info "Цели модуля"
    После прохождения этого модуля вы будете:

    - понимать и использовать workspaces для monorepo-проектов;
    - организовывать структуру multi-package проекта;
    - управлять зависимостями между пакетами внутри workspace.

---

## Практическое задание

### Задание: Создание workspace

1. Создайте директорию `practice-workspace/`.
2. В корневом `pyproject.toml` объявите workspace с двумя пакетами: `core` и
   `cli`.
3. Пакет `core` должен содержать модуль с функцией `greet(name: str) -> str`.
4. Пакет `cli` должен зависеть от `core` и использовать функцию `greet`.
5. Выполните `uv sync` и проверьте, что `uv run --package cli python -c "from
   core import greet; print(greet('World'))"` работает.

---

## Контрольные вопросы

??? question "Как определяются участники workspace?"
    В секции `[tool.uv.workspace]` корневого `pyproject.toml` через параметр
    `members`, который принимает список glob-паттернов (например,
    `["packages/*"]`). Каждый пакет-участник должен содержать свой
    `pyproject.toml`.
