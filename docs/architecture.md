## 16. Архитектура слоёв

### 16.1 Принцип

Каждый слой знает только о слое ниже себя.
Бизнес-логика живёт **только** в `service`.
Никогда не обращайся к БД напрямую из роутера или хэндлера.

```
router / handler   ← принимает запрос, отдаёт ответ
      ↓
   service         ← бизнес-логика, валидация, правила
      ↓
  repository       ← только работа с БД, никакой логики
      ↓
    model          ← структура данных
```

### 16.2 Пример: FastAPI

```python
# models.py
from sqlalchemy.orm import Mapped, mapped_column
from myapp.database import Base


class User(Base):
    __tablename__ = 'users'

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True)
    hashed_password: Mapped[str]
    is_active: Mapped[bool] = mapped_column(default=True)
```

```python
# repositories.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from myapp.models import User


class UserRepository:
    """Только CRUD — никакой бизнес-логики."""

    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def get_by_email(
        self, email: str
    ) -> User | None:
        result = await self._session.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def create(
        self,
        email: str,
        hashed_password: str,
    ) -> User:
        user = User(
            email=email,
            hashed_password=hashed_password,
        )
        self._session.add(user)
        await self._session.flush()
        return user
```

```python
# services.py
from typing import Protocol

from myapp.models import User
from myapp.security import hash_password, verify_password
from myapp.exceptions import (
    AlreadyExistsError,
    InvalidCredentialsError,
)


class UserRepo(Protocol):
    """Интерфейс репозитория (см. 6.4 Protocol)."""

    async def get_by_email(
        self, email: str,
    ) -> User | None: ...

    async def create(
        self, email: str, hashed_password: str,
    ) -> User: ...


class UserService:
    """Бизнес-логика — не знает про HTTP и не знает про SQL."""

    def __init__(self, repo: UserRepo) -> None:
        self._repo = repo

    async def register(
        self, email: str, password: str
    ) -> User:
        existing = await self._repo.get_by_email(email)
        if existing:
            raise AlreadyExistsError(
                f'Email {email!r} уже занят.'
            )
        hashed = hash_password(password)
        return await self._repo.create(email, hashed)

    async def authenticate(
        self, email: str, password: str
    ) -> User:
        user = await self._repo.get_by_email(email)
        if not user or not verify_password(
            user.hashed_password, password
        ):
            raise InvalidCredentialsError(
                'Неверный email или пароль.'
            )
        return user
```

```python
# routers/users.py
from fastapi import APIRouter, Depends, status
from myapp.services import UserService
from myapp.schemas import UserCreate, UserResponse
from myapp.dependencies import get_user_service

router = APIRouter(prefix='/users', tags=['users'])


@router.post(
    '/register',
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED,
)
async def register(
    data: UserCreate,
    service: UserService = Depends(get_user_service),
) -> UserResponse:
    """Роутер только принимает и отдаёт — логики нет."""
    user = await service.register(
        email=data.email,
        password=data.password,
    )
    return UserResponse.model_validate(user)
```

### 16.3 Пример: Django

В Django роль `repository` выполняет `Manager` модели,
но лучше создавать явный сервисный слой:

```python
# services.py
from collections.abc import Sequence
from typing import TypedDict

from django.db import transaction
from myapp.models import Order, OrderItem, Product
from myapp.exceptions import InsufficientStockError


class OrderItemData(TypedDict):
    product: Product
    quantity: int


class OrderService:

    @transaction.atomic
    def create_order(
        self,
        user_id: int,
        items: Sequence[OrderItemData],
    ) -> Order:
        order = Order.objects.create(user_id=user_id)
        for item_data in items:
            product = item_data['product']
            qty = item_data['quantity']
            if product.stock < qty:
                raise InsufficientStockError(
                    f'Недостаточно товара: {product.name}.'
                )
            product.stock -= qty
            product.save(update_fields=['stock'])
            OrderItem.objects.create(
                order=order,
                product=product,
                quantity=qty,
            )
        return order
```

### 16.4 Структура файлов

```
myapp/
├── routers/        # или handlers/ для aiogram
│   └── users.py
├── services/
│   └── user_service.py
├── repositories/
│   └── user_repo.py
├── models.py
├── schemas.py      # Pydantic-схемы (FastAPI)
├── exceptions.py
├── dependencies.py # DI-контейнер (FastAPI)
└── constants.py
```

---
