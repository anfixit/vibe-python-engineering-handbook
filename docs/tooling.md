## Управление зависимостями

### `pyproject.toml` vs `requirements.txt`

`pyproject.toml` — современный стандарт (PEP 517/518/621).
`requirements.txt` — легаси, которое живёт только потому,
что его все знают.

| | `pyproject.toml` | `requirements.txt` |
|---|---|---|
| Стандарт | PEP 621, актуален | Нет стандарта |
| Инструменты | `uv`, `poetry`, `hatch` | `pip` |
| Конфиги (`ruff`, `mypy`) | Да, в одном файле | Нет |
| Lock-файл | `uv.lock` / `poetry.lock` | Нужен `pip-tools` |
| Использовать когда | Всегда для нового | Только если среда требует |

**Используй `pyproject.toml` для всего нового.**
`requirements.txt` оставляй только если деплоишь
в среду без выбора (старый CI, хостинг).

### `uv` — современный менеджер пакетов

`uv` — замена `pip` + `venv` + `pip-tools`, написанная
на Rust. Работает в 10–100 раз быстрее `pip`.

```bash
# Установка
curl -LsSf https://astral.sh/uv/install.sh | sh

# Новый проект
uv init myproject
cd myproject

# Закрепить версию Python
# (uv run будет использовать именно её)
echo '3.12' > .python-version

# Добавить зависимость (обновляет pyproject.toml + uv.lock)
uv add aiogram pydantic-settings argon2-cffi

# Добавить dev-зависимость
uv add --dev pytest ruff mypy

# Установить всё из lock-файла (для деплоя)
uv sync

# ❗ uv.lock ОБЯЗАТЕЛЬНО коммить в git —
# он фиксирует точные версии для воспроизводимых сборок.
# Без него uv sync на сервере поставит другие версии.

# Запустить команду в окружении
uv run python -m myapp
uv run pytest
```

### Структура `pyproject.toml`

```toml
[project]
name = "myapp"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "aiogram>=3.0",
    "pydantic-settings>=2.0",
    "argon2-cffi>=23.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-cov>=5.0",
    "ruff>=0.4",
    "mypy>=1.10",
]

[tool.ruff]
line-length = 79
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "W", "I", "N", "UP", "S", "B"]
ignore = ["S101"]  # assert в тестах допустим

[tool.ruff.lint.isort]
force-single-line = false

[tool.mypy]
python_version = "3.12"
strict = true  # для новых проектов; иначе включай по шагам
ignore_missing_imports = true

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--cov=src --cov-report=term-missing"
```

Один файл вместо трёх (`setup.py` + `setup.cfg`
+ `requirements.txt` + отдельных конфигов для каждого
инструмента).

### Когда всё же нужен `requirements.txt`

Если среда требует именно его — генерируй из lock-файла,
не пиши вручную:

```bash
# Из uv
uv export --no-hashes > requirements.txt

# Из poetry
poetry export -f requirements.txt --output requirements.txt
```

Это гарантирует, что файл синхронизирован с реальными
зависимостями, а не устарел.

---

## Инструменты (обязательный набор)

### Установка через uv

```bash
# Основные зависимости
uv add pydantic-settings argon2-cffi

# Dev-зависимости
uv add --dev ruff mypy pytest pytest-cov pre-commit
```

Конфигурация всех инструментов — в `pyproject.toml`
(см. раздел 14.3). Никаких отдельных `.flake8`, `mypy.ini`,
`setup.cfg` — всё в одном файле.

### Pre-commit хук

```bash
uv run pre-commit install
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

После установки `ruff` запускается автоматически
перед каждым коммитом. Не пройдёт линтер — коммит
не создастся.

Примечание: `ruff-format` учитывает `line-length` из `[tool.ruff]`.

**Примечание:** `mypy` в pre-commit слишком медленный
для больших проектов. Лучше запускать его отдельно в CI
или вручную перед пушем:

```bash
# Локально
uv run mypy src/

# В CI (GitHub Actions)
- name: Type check
  run: uv run mypy src/
```

---
