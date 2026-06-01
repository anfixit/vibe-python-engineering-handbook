## 22. HTTP-клиенты (httpx)

### 22.1 Один клиент на всё приложение

```python
# Не создавай клиент в каждой функции — утечка соединений

# ❌
async def get_user(user_id: int) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(f'{BASE_URL}/users/{user_id}')
        return response.json()


# ✅ Клиент создаётся один раз при старте приложения
# main.py / lifespan
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http_client = httpx.AsyncClient(
        base_url=settings.external_api_url,
        timeout=httpx.Timeout(
            connect=5.0,
            read=30.0,
            write=10.0,
            pool=5.0,
        ),
        limits=httpx.Limits(
            max_connections=100,
            max_keepalive_connections=20,
        ),
    )
    yield
    await app.state.http_client.aclose()


# Dependency для роутеров
async def get_http_client(
    request: Request,
) -> httpx.AsyncClient:
    return request.app.state.http_client
```

### 22.2 Таймауты — всегда, явно

```python
# ❌ Без таймаута — может висеть вечно
response = await client.get(url)

# ✅ Явные таймауты
timeout = httpx.Timeout(
    connect=5.0,   # время на установку соединения
    read=30.0,     # время на чтение ответа
    write=10.0,    # время на отправку запроса
    pool=5.0,      # время ожидания свободного соединения из пула
)
response = await client.get(url, timeout=timeout)
```

### 22.3 Retry с exponential backoff

```python
# uv add tenacity
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type,
)
import httpx


@retry(
    stop=stop_after_attempt(MAX_RETRY_COUNT),
    wait=wait_exponential(
        multiplier=RETRY_BACKOFF_FACTOR,
        min=1,
        max=60,
    ),
    retry=retry_if_exception_type(
        (httpx.ConnectError, httpx.TimeoutException)
    ),
)
async def fetch_with_retry(
    client: httpx.AsyncClient,
    url: str,
) -> dict:
    response = await client.get(url)
    response.raise_for_status()
    return response.json()
```

### 22.4 Обработка ошибок HTTP

```python
from collections.abc import Mapping
from typing import Any


async def call_external_api(
    client: httpx.AsyncClient,
    endpoint: str,
    payload: Mapping[str, Any],
) -> dict:
    """Вызвать внешний API с полной обработкой ошибок."""
    try:
        response = await client.post(endpoint, json=payload)
        response.raise_for_status()
        return response.json()

    except httpx.TimeoutException:
        logger.error('Таймаут запроса к %s', endpoint)
        raise ServiceUnavailableError(
            f'Сервис {endpoint!r} не ответил вовремя.'
        )
    except httpx.HTTPStatusError as exc:
        logger.error(
            'HTTP %s от %s: %s',
            exc.response.status_code,
            endpoint,
            exc.response.text,
        )
        if exc.response.status_code == 404:
            raise NotFoundError('Ресурс не найден.')
        raise ExternalServiceError(
            f'Ошибка API: {exc.response.status_code}.'
        )
    except httpx.RequestError as exc:
        logger.error('Сетевая ошибка: %s', exc)
        raise ServiceUnavailableError('Сетевая ошибка.')
```

### 22.5 Проверка ответа

```python
# Всегда проверяй структуру ответа перед использованием
from pydantic import BaseModel, ValidationError


class ExternalUserResponse(BaseModel):
    id: int
    email: str
    name: str


async def get_external_user(user_id: int) -> ExternalUserResponse:
    data = await fetch_with_retry(client, f'/users/{user_id}')

    try:
        return ExternalUserResponse.model_validate(data)
    except ValidationError as exc:
        logger.error(
            'Неожиданная структура ответа API: %s', exc
        )
        raise ExternalServiceError(
            'API вернул неожиданный формат данных.'
        )
```

---
