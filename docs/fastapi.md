## 11. FastAPI специфика

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI, Depends, HTTPException, status


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Инициализация и завершение работы."""
    validate_settings()   # падаем громко при старте
    await db.connect()
    yield
    await db.disconnect()


app = FastAPI(lifespan=lifespan)


# Dependency Injection для общих зависимостей
async def get_current_user(
    token: str = Depends(oauth2_scheme),
) -> User:
    """Получить текущего пользователя из токена."""
    user = await auth_service.verify_token(token)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail='Недействительный токен.',
        )
    return user
```

### 11.1 Документация API — автогенерация

FastAPI генерирует OpenAPI (Swagger / ReDoc) автоматически.
Но качество документации зависит от того, как ты описываешь
эндпоинты:

```python
@router.post(
    '/users',
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED,
    summary='Регистрация нового пользователя',
    description=(
        'Создаёт пользователя, хэширует пароль, '
        'отправляет email подтверждения.'
    ),
    responses={
        409: {'description': 'Email уже зарегистрирован'},
        422: {'description': 'Ошибка валидации'},
    },
)
async def register(data: UserCreate) -> UserResponse:
    ...
```

Правила:

- Каждый эндпоинт — `summary` (краткое) и `description`.
- `response_model` — всегда явно.
- `responses` — описывай все нестандартные статусы.
- Примеры запросов/ответов — через `model_config`
  в Pydantic-схемах:

```python
from pydantic import BaseModel, ConfigDict, EmailStr, Field


class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8)

    model_config = ConfigDict(
        json_schema_extra={
            'examples': [
                {
                    'email': 'user@example.com',
                    'password': 'SecurePass1',
                }
            ]
        }
    )
```

---
