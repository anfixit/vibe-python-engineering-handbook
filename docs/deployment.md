## Деплой и продакшн

### Чеклист перед деплоем

- [ ] Секреты: `.env` не в git, значения только из env/.env (см. 3)
- [ ] `DEBUG=false` в продакшне (см. 3.3)
- [ ] Пароли: Argon2id/bcrypt/scrypt, не MD5/SHA (см. 8.1)
- [ ] Валидация входных данных на всех входах (см. 8.2)
- [ ] Auth endpoints защищены rate limiting (см. 8.4)
- [ ] CORS без `*` в продакшне, только явные домены (см. 8.5)
- [ ] CSP и security headers включены (см. 8.6)
- [ ] Куки: HttpOnly, Secure, SameSite (см. 8.7)
- [ ] JWT без чувствительных данных (см. 8.3)
- [ ] RLS включён где уместно (см. 8.8)
- [ ] `pip-audit` / `uv run pip-audit` без критических CVE (см. 8.9)
- [ ] Логи без секретов и токенов (см. 7.3)
- [ ] Health check работает, мониторинг подключён (см. 23)
- [ ] Graceful shutdown настроен (см. 17.7)
- [ ] Смоук-тест на проде: регистрация, логин, ключевые сценарии

### Docker

```dockerfile
# Dockerfile
FROM python:3.12-slim

# Без .pyc в контейнере, без буферизации stdout
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Не запускаем от root
RUN useradd -m appuser
WORKDIR /app

# Установка uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Зависимости отдельным слоем (кэш)
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY --chown=appuser:appuser . .
USER appuser

CMD ["uv", "run", "python", "-m", "myapp"]
```

```yaml
# docker-compose.yml
services:
  app:
    build: .
    env_file: .env          # ← секреты из файла
    environment:
      - DEBUG=false
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    env_file: .env
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U $$POSTGRES_USER']
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  pgdata:
```

**Никогда** не хардкодь пароли в `docker-compose.yml`.
`env_file: .env` ссылается на локальный файл —
убедись, что `.env` есть в `.gitignore` (см. раздел 3.2)
и **никогда** не попадает в git.

### Systemd-сервис

```ini
# /etc/systemd/system/mybot.service
[Unit]
Description=My Telegram Bot
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/mybot
EnvironmentFile=/opt/mybot/.env
ExecStart=/usr/local/bin/uv run python -m myapp
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable mybot
sudo systemctl start mybot
sudo journalctl -u mybot -f  # логи в реальном времени
```

Секреты: `EnvironmentFile=/opt/mybot/.env` — systemd читает
файл и передаёт переменные процессу. Файл должен быть
доступен только `appuser`: `chmod 600 /opt/mybot/.env`.

### GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install uv
        run: curl -LsSf https://astral.sh/uv/install.sh | sh

      - name: Install dependencies
        run: uv sync

      - name: Run tests
        run: uv run pytest --tb=short

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/mybot
            git pull origin main
            uv sync
            sudo systemctl restart mybot
```

Секреты (`SERVER_HOST`, `SSH_PRIVATE_KEY`) хранятся
в **GitHub → Settings → Secrets and variables → Actions**.
Никогда не в коде и не в `yml`.

---
