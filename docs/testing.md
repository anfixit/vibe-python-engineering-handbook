## Тестирование: философия и pytest

### Пирамида тестов

```
        /\
       /  \   E2E тесты (мало, дорого, медленно)
      /----\
     /      \ Интеграционные (БД, внешние сервисы)
    /--------\
   /          \ Юнит-тесты (много, быстро, дёшево)
  /____________\
```

Большинство тестов — **юниты**. Они тестируют одну функцию
в изоляции, без БД и сети.

### Что тестировать

```
✅ Бизнес-логику в service-слое
✅ Граничные условия (пустой список, None, 0, максимум)
✅ Сценарии ошибок (что падает правильно)
✅ Публичный интерфейс функций

❌ Не тестируй реализацию — тестируй поведение
❌ Не тестируй чужой код (Django ORM, Pydantic)
❌ Не тестируй тривиальные геттеры/сеттеры
```

### Структура теста: AAA

**Arrange — Act — Assert.** Три чётких блока.

```python
def test_calculate_discount_returns_correct_price():
    # Arrange
    price = Decimal('1000')
    percent = Decimal('20')

    # Act
    result = calculate_discount(price, percent)

    # Assert
    assert result == Decimal('800')


def test_calculate_discount_raises_on_invalid_percent():
    # Arrange
    price = Decimal('1000')
    invalid_percent = Decimal('150')

    # Act & Assert
    with pytest.raises(ValueError, match='0–100'):
        calculate_discount(price, invalid_percent)
```

### Фикстуры вместо хардкода

```python
# conftest.py
import pytest
from myapp.models import User


@pytest.fixture
def active_user() -> User:
    return User(
        id=1,
        email='test@example.com',
        is_active=True,
        role=Role.USER,
    )


@pytest.fixture
def admin_user() -> User:
    return User(
        id=2,
        email='admin@example.com',
        is_active=True,
        role=Role.ADMIN,
    )
```

### Фабрики для сложных объектов

```python
# conftest.py
from dataclasses import dataclass, field


@dataclass
class UserFactory:
    """Фабрика тестовых пользователей."""

    id: int = 1
    email: str = 'user@example.com'
    role: Role = Role.USER
    is_active: bool = True

    def build(self) -> User:
        return User(**self.__dict__)


@pytest.fixture
def make_user():
    def _factory(**kwargs) -> User:
        return UserFactory(**kwargs).build()
    return _factory


# В тесте
def test_admin_can_delete(make_user):
    admin = make_user(role=Role.ADMIN)
    user = make_user(id=2, email='other@example.com')
    ...
```

### Мокирование

```python
from unittest.mock import AsyncMock, MagicMock, patch


# Мокировать внешний HTTP-вызов
@patch('myapp.services.http_client.get')
async def test_fetch_user_profile(mock_get):
    mock_get.return_value = AsyncMock(
        status_code=200,
        json=lambda: {'id': 1, 'name': 'Test'},
    )
    result = await fetch_user_profile(user_id=1)
    assert result.name == 'Test'


# Мокировать репозиторий целиком
async def test_register_raises_if_email_taken():
    repo = MagicMock()
    repo.get_by_email = AsyncMock(
        return_value=User(email='taken@example.com')
    )
    service = UserService(repo=repo)

    with pytest.raises(AlreadyExistsError):
        await service.register(
            email='taken@example.com',
            password='SecurePass1',
        )
```

### Параметризация

```python
@pytest.mark.parametrize('percent,expected', [
    (Decimal('0'), Decimal('1000')),
    (Decimal('50'), Decimal('500')),
    (Decimal('100'), Decimal('0')),
    (Decimal('20'), Decimal('800')),
])
def test_discount_parametrized(percent, expected):
    assert calculate_discount(
        Decimal('1000'), percent
    ) == expected


@pytest.mark.parametrize('invalid', [
    Decimal('-1'), Decimal('101'), Decimal('200'),
])
def test_discount_invalid_percent(invalid):
    with pytest.raises(ValueError):
        calculate_discount(Decimal('1000'), invalid)
```

### Pytest конфиг и полезные плагины

```bash
uv add --dev pytest pytest-cov pytest-asyncio pytest-mock
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"   # все async-тесты без @pytest.mark.asyncio
addopts = [
    "--cov=src",
    "--cov-report=term-missing",
    "--cov-fail-under=80",  # упасть если покрытие < 80%
]
```

---
