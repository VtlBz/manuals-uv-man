# uv - Guide for Python Developers

> [Русская версия](README.md)

This repository contains source materials for a practical guide to
[uv](https://docs.astral.sh/uv/) - a modern package and project manager
for Python by [Astral](https://astral.sh/).

The guide covers working with `uv` from installation and basic commands
to advanced scenarios: Python version management, dependencies,
environments, sync workflow, Docker/CI, workspaces, package publishing.
The content is written in Russian, formatted in Markdown, and built into
a static site via MkDocs Material for publishing on GitHub Pages.

In addition to the guide, the repository stores supplementary materials
for creating an educational course on `uv` (practical assignments,
self-check questions).

## Target Audience

Python developers who:

- use `requirements.txt`, `pip-tools`, `pyenv`, `poetry`
  and want to switch to `uv`;
- are looking for a single tool for managing Python projects.

## Repository Structure

`docs/` contains the main guide - 15 sections in Markdown format.
These files are built into a static site and published on GitHub Pages.

`course/` stores supplementary materials for creating an educational
course based on the guide: practical assignments and self-check questions,
grouped by section.

Other files are project configuration:

```text
mkdocs.yml      MkDocs Material configuration
pyproject.toml  project dependencies (uv-managed)
uv.lock         dependency lockfile
```

## Local Build

The project is managed via `uv`. To build and preview locally:

```bash
uv sync
uv run mkdocs serve
```

The site will be available at `http://127.0.0.1:8000/`.

For a static build:

```bash
uv run mkdocs build
```

## Deployment

The site is published via GitHub Pages using GitHub Actions.
A push to `main` triggers an automatic build and deploy.

## License

Documentation (`docs/`, `course/`) - [CC BY-NC-SA 4.0](LICENSE-CC).
All other files - [BSD 3-Clause](LICENSE-BSD).
