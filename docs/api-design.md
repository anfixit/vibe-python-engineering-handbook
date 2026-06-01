## 24. Стандартизация ответов API

### 24.1 Единый формат ответов

Клиент не должен угадывать структуру ответа.
Успехи и ошибки — в одном предсказуемом формате.

**Успешные ответы:**

```python
# schemas/responses.py
from typing import Any, Generic, TypeVar
from pydantic import BaseModel

T = TypeVar('T')


class PaginationMeta(BaseModel):
    page: int
    per_page: int
    total: int
    total_pages: int


class ApiResponse(BaseModel, Generic[T]):
    """Обёртка для всех успешных ответов."""
    data: T
    meta: dict[str, Any] = {}
```

Примеры ответов:

```json
// Один объект
{
    "data": {"id": 1, "email": "user@example.com"},
    "meta": {}
}

// Список с пагинацией
{
    "data": [
        {"id": 1, "email": "user@example.com"},
        {"id": 2, "email": "admin@example.com"}
    ],
    "meta": {
        "page": 1,
        "per_page": 20,
        "total": 42,
        "total_pages": 3
    }
}
```

**Использование в роутерах:**

```python
@router.get('/users', response_model=ApiResponse[list[UserResponse]])
async def list_users(
    page: int = Query(1, ge=1),
    per_page: int = Query(DEFAULT_PAGE_SIZE, ge=1, le=MAX_PAGE_SIZE),
    service: UserService = Depends(get_user_service),
) -> ApiResponse[list[UserResponse]]:
    users, total = await service.list_users(page, per_page)
    return ApiResponse(
        data=[UserResponse.model_validate(u) for u in users],
        meta=PaginationMeta(
            page=page,
            per_page=per_page,
            total=total,
            total_pages=-(-total // per_page),
        ).model_dump(),
    )
```

**Ошибки:**

```python
# schemas/errors.py
from pydantic import BaseModel


class ErrorDetail(BaseModel):
    field: str | None = None
    message: str


class ErrorResponse(BaseModel):
    code: str
    message: str
    details: list[ErrorDetail] = []
```

Примеры ответов:

```json
// 400
{
    "code": "VALIDATION_ERROR",
    "message": "Ошибка валидации входных данных.",
    "details": [
        {"field": "email", "message": "Невалидный email."},
        {"field": "password", "message": "Минимум 8 символов."}
    ]
}

// 404
{
    "code": "NOT_FOUND",
    "message": "Пользователь с id=42 не найден.",
    "details": []
}

// 409
{
    "code": "ALREADY_EXISTS",
    "message": "Email уже зарегистрирован.",
    "details": []
}
```

### 24.2 Exception handler в FastAPI

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse


@app.exception_handler(AppError)
async def app_error_handler(
    request: Request,
    exc: AppError,
) -> JSONResponse:
    """Единый обработчик бизнес-исключений."""
    error_map = {
        ValidationError: (400, 'VALIDATION_ERROR'),
        NotFoundError: (404, 'NOT_FOUND'),
        AlreadyExistsError: (409, 'ALREADY_EXISTS'),
        InvalidCredentialsError: (401, 'INVALID_CREDENTIALS'),
        ServiceUnavailableError: (503, 'SERVICE_UNAVAILABLE'),
    }

    status_code, code = error_map.get(
        type(exc), (500, 'INTERNAL_ERROR')
    )

    return JSONResponse(
        status_code=status_code,
        content={
            'code': code,
            'message': str(exc),
            'details': [],
        },
    )
```

### 24.3 API versioning

Для API с внешними клиентами — версионируй через
URL prefix:

```python
# routers/v1/users.py
router = APIRouter(prefix='/api/v1/users')

# routers/v2/users.py
router = APIRouter(prefix='/api/v2/users')

# main.py
app.include_router(v1_router)
app.include_router(v2_router)
```

Правила:

- Новая версия — только при **ломающих** изменениях.
- Старую версию поддерживай минимум 6 месяцев.
- Добавление полей в ответ — **не** ломающее изменение.
- Удаление или переименование полей — ломающее.

### 24.4 Pagination — реализация

Константы `DEFAULT_PAGE_SIZE` и `MAX_PAGE_SIZE`
определены в `constants.py`. Вот как их использовать.

**Offset-based (простая, для большинства случаев):**

```python
# repositories/user_repo.py
from sqlalchemy import select, func


class UserRepository:

    async def list_paginated(
        self,
        page: int,
        per_page: int,
    ) -> tuple[list[User], int]:
        """Вернуть страницу и общее количество."""
        # Общее количество
        total = await self._session.scalar(
            select(func.count(User.id))
        )

        # Страница
        offset = (page - 1) * per_page
        result = await self._session.execute(
            select(User)
            .order_by(User.id)
            .offset(offset)
            .limit(per_page)
        )
        users = list(result.scalars().all())

        return users, total
```

```python
# Django эквивалент
from django.core.paginator import Paginator

def list_users(page: int, per_page: int):
    qs = User.objects.all().order_by('id')
    paginator = Paginator(qs, per_page)
    page_obj = paginator.page(page)
    return page_obj.object_list, paginator.count
```

**Cursor-based (для больших таблиц и бесконечного скролла):**

Offset-pagination становится медленной на больших таблицах
(`OFFSET 100000` всё равно сканирует 100000 строк).
Cursor-based решает это:

```python
async def list_after_cursor(
    self,
    cursor_id: int | None,
    limit: int,
) -> list[User]:
    """Cursor-based: быстрая на любом объёме."""
    query = select(User).order_by(User.id).limit(limit)
    if cursor_id is not None:
        query = query.where(User.id > cursor_id)
    result = await self._session.execute(query)
    return list(result.scalars().all())
```

**Валидация параметров пагинации:**

```python
from fastapi import Query


@router.get('/users')
async def list_users(
    page: int = Query(1, ge=1, description='Номер страницы'),
    per_page: int = Query(
        DEFAULT_PAGE_SIZE,
        ge=1,
        le=MAX_PAGE_SIZE,
        description='Записей на странице',
    ),
) -> ApiResponse[list[UserResponse]]:
    ...
```

Правила:

- **Всегда** ограничивай `per_page` сверху (`le=MAX_PAGE_SIZE`).
- **Всегда** задавай `ORDER BY` — без него порядок
  не гарантирован и страницы будут «прыгать».
- Offset — для админ-панелей, таблиц с номерами страниц.
- Cursor — для лент, бесконечного скролла, таблиц > 100K строк.

---

## 26. Идемпотентность

### 26.1 Зачем

Идемпотентная операция даёт одинаковый результат
при повторном выполнении. Это критично когда:

- Клиент получил таймаут и отправил запрос повторно.
- Telegram доставил update дважды.
- Платёжная система сделала retry webhook'а.

Без идемпотентности — двойное списание, двойная
регистрация, дублирование заказов.

### 26.2 Идемпотентные ключи в API

```python
# Клиент отправляет уникальный ключ в заголовке:
# X-Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

from fastapi import Header


@router.post('/payments')
async def create_payment(
    data: PaymentCreate,
    idempotency_key: str = Header(
        alias='X-Idempotency-Key',
    ),
    service: PaymentService = Depends(get_payment_service),
) -> PaymentResponse:
    return await service.create_payment(
        data=data,
        idempotency_key=idempotency_key,
    )
```

```python
# services/payment_service.py
class PaymentService:

    async def create_payment(
        self,
        data: PaymentCreate,
        idempotency_key: str,
    ) -> Payment:
        # 1. Проверяем — уже обработан?
        existing = await self._repo.get_by_idempotency_key(
            idempotency_key
        )
        if existing:
            return existing  # возвращаем тот же результат

        # 2. Выполняем операцию
        payment = await self._repo.create(
            amount=data.amount,
            idempotency_key=idempotency_key,
        )
        return payment
```

```sql
-- Уникальный индекс в БД — последний рубеж
CREATE UNIQUE INDEX ix_payments_idempotency
ON payments (idempotency_key);
```

### 26.3 Идемпотентность в Telegram-ботах

Telegram может доставить один update несколько раз
(особенно при webhook). Защита:

```python
# middlewares/dedup.py
from cachetools import TTLCache
from aiogram import BaseMiddleware
from aiogram.types import Update

# Хранить ID обработанных update'ов 5 минут
_processed: TTLCache = TTLCache(maxsize=50_000, ttl=300)


class DeduplicationMiddleware(BaseMiddleware):
    """Игнорировать повторные update'ы."""

    async def __call__(self, handler, event, data):
        if isinstance(event, Update):
            if event.update_id in _processed:
                return  # уже обработали
            _processed[event.update_id] = True
        return await handler(event, data)
```

### 26.4 Какие операции должны быть идемпотентными

```
✅ Обязательно идемпотентны:
- Создание платежа / заказа
- Webhook-обработчики (Telegram, Stripe, любые)
- Любая операция, где retry возможен
- POST-запросы, создающие ресурсы

Естественно идемпотентны (не нужен ключ):
- GET, HEAD, OPTIONS — по определению
- PUT (полное обновление) — результат одинаков
- DELETE — удаление уже удалённого = ok

⚠️ Не идемпотентны по умолчанию (нужен ключ):
- POST (создание)
- PATCH (инкрементные изменения: balance += 100)
```

---
