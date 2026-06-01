## Health checks и мониторинг

### Health check эндпоинт

Для Docker, Kubernetes и мониторинга — обязательный
эндпоинт, который проверяет состояние сервиса:

```python
# routers/health.py
from typing import TypedDict

from fastapi import APIRouter, status
from fastapi.responses import JSONResponse

router = APIRouter(tags=['health'])


class HealthResponse(TypedDict):
    status: str


@router.get('/health')
async def health() -> HealthResponse:
    """Liveness probe: сервис запущен."""
    return {'status': 'ok'}


@router.get('/ready')
async def readiness(
    session: AsyncSession = Depends(get_session),
) -> JSONResponse:
    """Readiness probe: сервис готов принимать трафик."""
    checks = {}

    # Проверка БД
    try:
        await session.execute(text('SELECT 1'))
        checks['database'] = 'ok'
    except Exception as exc:
        checks['database'] = f'error: {exc}'

    # Проверка Redis (если используется)
    try:
        await redis.ping()
        checks['redis'] = 'ok'
    except Exception as exc:
        checks['redis'] = f'error: {exc}'

    all_ok = all(v == 'ok' for v in checks.values())
    return JSONResponse(
        status_code=(
            status.HTTP_200_OK
            if all_ok
            else status.HTTP_503_SERVICE_UNAVAILABLE
        ),
        content={'status': 'ok' if all_ok else 'degraded',
                 'checks': checks},
    )
```

### Sentry — error tracking

Sentry автоматически ловит необработанные исключения,
группирует их и уведомляет. Для продакшна — обязательно.

```python
# uv add 'sentry-sdk[fastapi]'
import sentry_sdk

if settings.sentry_dsn:
    sentry_sdk.init(
        dsn=settings.sentry_dsn.get_secret_value(),
        traces_sample_rate=0.1,  # 10% трейсов
        environment='production',
        # Не отправляй чувствительные данные
        send_default_pii=False,
    )
```

Для Django:

```python
# uv add 'sentry-sdk[django]'
import sentry_sdk

if settings.sentry_dsn:
    sentry_sdk.init(
        dsn=settings.sentry_dsn.get_secret_value(),
        integrations=[
            sentry_sdk.integrations.django.DjangoIntegration(),
        ],
        traces_sample_rate=0.1,
    )
```

Для Aiogram-бота:

```python
# В main.py, до создания бота
import sentry_sdk

if settings.sentry_dsn:
    sentry_sdk.init(
        dsn=settings.sentry_dsn.get_secret_value(),
    )
```

Правила:

- Подключай Sentry **до** создания app/bot/dispatcher.
- `send_default_pii=False` — не отправляй IP, email, куки.
- `traces_sample_rate` — 0.1 (10%) достаточно для начала.
- В `.env.example` добавь `SENTRY_DSN=`.

---
