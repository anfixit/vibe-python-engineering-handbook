## Git: конвенция коммитов и ветки

### Conventional Commits

Формат: `<тип>(<scope>): <описание>`

```
feat(auth): добавить регистрацию через email
fix(bot): исправить падение при пустом сообщении
docs(readme): обновить инструкцию по запуску
refactor(users): вынести логику в UserService
test(orders): добавить тест граничных условий
chore(deps): обновить aiogram до 3.10
perf(db): добавить индекс на users.email
```

Типы:

| Тип | Когда |
|---|---|
| `feat` | Новая функциональность |
| `fix` | Исправление бага |
| `refactor` | Рефакторинг без изменения поведения |
| `test` | Тесты |
| `docs` | Документация |
| `chore` | Зависимости, конфиги, инфраструктура |
| `perf` | Оптимизация производительности |
| `ci` | CI/CD |

### Модель веток

```
main          ← только стабильный код, прямые пуши запрещены
  └── develop ← интеграционная ветка (опционально)
        ├── feature/user-registration
        ├── feature/telegram-notifications
        ├── fix/empty-message-crash
        └── chore/update-dependencies
```

Именование веток: `<тип>/<короткое-описание-через-дефис>`

### Правила работы с git

```bash
# ✅ Создавай ветку от main
git checkout main && git pull
git checkout -b feature/add-payments

# ✅ Маленькие атомарные коммиты по ходу работы
git add -p   # добавляй кусками, не весь файл скопом
git commit -m "feat(payments): добавить схему PaymentCreate"

# ✅ Перед PR — rebase, не merge
git fetch origin
git rebase origin/main

# ✅ Squash мусорных коммитов перед merge
git rebase -i origin/main   # объединить "fix typo" коммиты
```

### Что никогда не делать

```
❌ Коммитить в main напрямую
❌ git push --force на общих ветках
❌ Коммиты "fix", "wip", "asdf" без описания
❌ Один огромный коммит на всю фичу
❌ Коммитить .env, секреты, *.pyc, __pycache__
❌ Коммитить закомментированный код
```

### `.gitignore` для Python-проекта

```gitignore
# Секреты — самое важное
.env
.env.*
!.env.example
*.pem
*.key

# Python
__pycache__/
*.py[cod]
*.egg-info/
.venv/
dist/
build/

# Инструменты
.mypy_cache/
.ruff_cache/
.pytest_cache/
.coverage
htmlcov/

# IDE
.idea/
.vscode/
*.swp

# OS
.DS_Store
Thumbs.db

# Логи
logs/
*.log
```

---
