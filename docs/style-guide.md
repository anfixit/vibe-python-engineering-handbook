## Философия

Код читают чаще, чем пишут. Каждое решение принимай
с вопросом: **«Поймёт ли это я через полгода?»**

Приоритеты (по убыванию важности):

1. Корректность
2. Безопасность
3. Читаемость
4. Производительность

**Минимальная версия Python: 3.12.** Примеры используют
`StrEnum`, синтаксис `type X | Y`, скобочный `with` и другие
возможности 3.10+/3.11+/3.12.

Безопасный, но некорректный код — это код, который
молча делает не то. Сначала убедись, что результат
правильный, потом — что он безопасен. При дилемме
«показать данные с ошибкой или не показать вообще» —
выбирай второе. Но не забывай: корректность — фундамент,
без которого безопасность не имеет смысла.

---

## PEP 8: обязательные правила

### Длина строки

Максимум **79 символов**. Это не рекомендация — это закон.

Для переноса используй скобки, а не `\`:

```python
# ✅ Правильно
result = (
    some_long_function_name(argument_one, argument_two)
    + another_function(argument_three)
)

# ❌ Неправильно
result = some_long_function_name(argument_one, argument_two) \
    + another_function(argument_three)
```

Длинные строки импортов:

```python
# ✅
from some.module import (
    ClassOne,
    ClassTwo,
    function_one,
)
```

### Отступы и пустые строки

- Отступ: **4 пробела** (никаких табов).
- Между функциями/классами верхнего уровня: **2 пустые строки**.
- Между методами внутри класса: **1 пустая строка**.
- После `class MyClass:`: **1 пустая строка** перед первым методом.
- Внутри функций пустые строки — только для разделения
  **логических блоков**, не больше одной подряд.

```python
class MyService:

    def method_one(self):
        ...

    def method_two(self):
        ...


def top_level_function():
    ...
```

### Пробелы

```python
# ✅
x = 1
result = func(a, b)
my_dict = {'key': value}
lst[1:3]

# ❌
x=1
result = func( a,b )
my_dict = { 'key' : value }
lst[ 1 : 3 ]
```

Вокруг операторов — **один** пробел с каждой стороны.
Перед `:` в аргументах типизации — **без пробела**:

```python
def greet(name: str, count: int = 1) -> str:
    ...
```

### Кавычки

- Обычные строки: **одинарные** `'text'`.
- Докстринги: **тройные двойные** `"""Текст."""`.
- Строки с апострофом внутри: двойные `"it's fine"`.
- Во всём файле — **единообразно**.

### Именование

| Что                | Стиль         | Пример                  |
|--------------------|---------------|-------------------------|
| Переменные         | `snake_case`  | `user_count`            |
| Функции/методы     | `snake_case`  | `get_user_by_id()`      |
| Классы             | `PascalCase`  | `UserService`           |
| Константы          | `UPPER_SNAKE` | `MAX_RETRY_COUNT`       |
| Приватные атрибуты | `_snake_case` | `_internal_cache`       |
| «Магические»       | `__dunder__`  | `__init__`, `__str__`   |

Правила именования:

- Имя отражает **смысл**, а не тип: `users`, а не `users_list`.
- Глагол для функций: `calculate_tax()`, а не `tax()`.
- Без лишних префиксов: `author`, а не `current_author`.
- Без лишних суффиксов: `note`, а не `note_object`.

---

## Структура проекта

### Стандартный layout

```
project/
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── constants.py      ← все константы
│       ├── config.py         ← pydantic-settings
│       ├── models.py
│       ├── services.py       ← бизнес-логика
│       ├── repositories.py   ← работа с БД
│       └── utils.py
├── tests/
├── .env                      ← НЕ в git
├── .env.example              ← в git, без значений
├── .python-version           ← «3.12» для uv
├── .gitignore
├── pyproject.toml
└── uv.lock                   ← в git (pinned deps)
```

### Организация импортов

Три группы, разделённые **пустой строкой**, в строгом порядке:

```python
# Стандартная библиотека
import os
import sys
from pathlib import Path

# Сторонние библиотеки
import httpx
from pydantic import BaseModel

# Модули проекта
from myapp.constants import MAX_RETRY_COUNT
from myapp.models import User
```

Правила:

- Внутри группы — **по алфавиту**.
- Никогда `from module import *`.
- Никаких неиспользуемых импортов.
- Сортируй автоматически через `ruff` (правило `I`,
  настройки — в `[tool.ruff.lint.isort]`).

### `__all__` для контроля публичного API

В каждом модуле с публичным API определяй `__all__`,
чтобы явно обозначить публичный интерфейс модуля.
Это помогает IDE, автодополнению и инструментам
статического анализа:

```python
# services/user_service.py
__all__ = ['UserService']


class UserService:
    ...


class _InternalHelper:
    """Не экспортируется."""
    ...
```

```python
# services/__init__.py
from myapp.services.user_service import UserService

__all__ = ['UserService']
```

Правила:

- `__all__` — в каждом `__init__.py` пакета.
- В модулях с несколькими публичными сущностями.
- Если модуль содержит только одну публичную функцию —
  `__all__` не обязателен, но желателен.

---

## Константы и магические числа

### Правило: ноль магических чисел в коде

Магическое число/строка — любое литеральное значение,
смысл которого не очевиден из контекста.

Примечание: тривиальные литералы вроде `0`, `1`, `None`,
`True`, `False` допустимы,
если они очевидны из контекста.

```python
# ❌ Что такое 20? Почему 3? Что значит 'admin'?
if len(title) > 20:
    ...
for _ in range(3):
    ...
if user.role == 'admin':
    ...

# ✅ Всё понятно без комментариев
if len(title) > MAX_TITLE_LENGTH:
    ...
for _ in range(MAX_RETRY_COUNT):
    ...
if user.role == Role.ADMIN:
    ...
```

### Файл `constants.py`

```python
# constants.py

# --- Ограничения полей ---
MAX_TITLE_LENGTH = 200
MAX_CONTENT_LENGTH = 10_000
MIN_PASSWORD_LENGTH = 8
MAX_USERNAME_LENGTH = 50

# --- Пагинация ---
DEFAULT_PAGE_SIZE = 20
MAX_PAGE_SIZE = 100

# --- Временны́е интервалы (в секундах) ---
SESSION_TIMEOUT = 3_600        # 1 час
CACHE_TTL_SHORT = 300          # 5 минут
CACHE_TTL_LONG = 86_400        # 24 часа
TOKEN_EXPIRY = 604_800         # 7 дней

# --- Rate limiting ---
MAX_LOGIN_ATTEMPTS = 5
LOGIN_LOCKOUT_SECONDS = 900    # 15 минут
RATE_LIMIT_PER_MINUTE = 60

# --- HTTP ---
HTTP_TIMEOUT_SECONDS = 30
MAX_RETRY_COUNT = 3
RETRY_BACKOFF_FACTOR = 2

# --- Telegram ---
MAX_MESSAGE_LENGTH = 4096
MAX_CAPTION_LENGTH = 1024

# --- Загрузка файлов ---
MAX_UPLOAD_SIZE_BYTES = 10 * 1024 * 1024  # 10 MB
ALLOWED_IMAGE_TYPES = frozenset({
    'image/jpeg',
    'image/png',
    'image/webp',
})
```

### Enum для категорий значений

Примечание: `StrEnum` доступен начиная с Python 3.11.
В этом стандарте предполагается Python 3.12
(см. `requires-python` в примере `pyproject.toml`).

```python
# constants.py
from enum import StrEnum


class Role(StrEnum):
    ADMIN = 'admin'
    USER = 'user'
    MODERATOR = 'moderator'


class OrderStatus(StrEnum):
    PENDING = 'pending'
    PROCESSING = 'processing'
    COMPLETED = 'completed'
    CANCELLED = 'cancelled'
```

Использование Enum вместо строк даёт автодополнение в IDE
и исключает опечатки.

Для HTTP-статусов **не создавай свой Enum** — используй
стандартную библиотеку:

```python
from http import HTTPStatus

if response.status_code == HTTPStatus.NOT_FOUND:
    ...

# HTTPStatus.OK → 200
# HTTPStatus.CREATED → 201
# HTTPStatus.BAD_REQUEST → 400
# и так далее — всё уже есть
```

---

## Python-сахар: когда использовать, а когда нет

### Списковые включения (list comprehensions)

Используй для **простых** преобразований (1–2 условия).

```python
# ✅ Читаемо
active_users = [u for u in users if u.is_active]
names = [u.name.upper() for u in users]

# ❌ Слишком сложно — лучше цикл
result = [
    transform(item)
    for item in collection
    if condition_one(item)
    if condition_two(item)
    for sub in item.children
]
```

Никогда не используй list comprehension для **побочных эффектов**:

```python
# ❌ ЗАПРЕЩЕНО — это читерство, не comprehension
[print(x) for x in items]
[db.save(obj) for obj in objects]

# ✅ Правильно
for x in items:
    print(x)
```

### Генераторные выражения

Предпочитай генераторы спискам, когда результат
используется **один раз**:

```python
# Генератор не создаёт весь список в памяти
total = sum(u.balance for u in users if u.is_active)
any_admin = any(u.role == Role.ADMIN for u in users)
first_match = next(
    (u for u in users if u.email == email),
    None,  # значение по умолчанию
)
```

### Словарные включения

```python
# ✅
user_map = {u.id: u for u in users}
scores = {name: score for name, score in zip(names, values)}

# Фильтрация словаря
active = {k: v for k, v in data.items() if v is not None}
```

### Walrus-оператор `:=` (Python 3.8+)

Используй **осторожно**, только когда убирает дублирование:

```python
# ✅ Оправданно — избегаем двойного вызова
if (user := get_user(user_id)) is not None:
    process(user)

# ✅ В цикле
while chunk := file.read(8192):
    process(chunk)

# ❌ Излишне усложняет
if (n := len(data)) > 10:  # просто напиши len(data) > 10
    ...
```

### f-строки

Используй f-строки для форматирования. Не конкатенацию.

```python
# ✅
message = f'Привет, {user.name}! У вас {count} сообщений.'
log_line = f'[{timestamp:%Y-%m-%d %H:%M}] {level}: {text}'

# ❌
message = 'Привет, ' + user.name + '!'
message = 'Привет, %s!' % user.name
```

Длинные f-строки разбивай:

```python
error_msg = (
    f'Ошибка валидации поля "{field_name}": '
    f'ожидалось {expected!r}, получено {actual!r}.'
)
```

### Распаковка (unpacking)

```python
# ✅ Распаковка кортежей
first, *rest = items
head, *middle, tail = items

# Swap без temp-переменной
a, b = b, a

# Распаковка в цикле
for user_id, username in users.items():
    ...

# Расширенная распаковка в функциях
def create_user(name: str, **kwargs: str) -> User:
    ...
```

### `any()` и `all()`

```python
# ✅ Вместо циклов с флагами
has_admin = any(u.role == Role.ADMIN for u in users)
all_active = all(u.is_active for u in users)

# ❌ Устаревший стиль
has_admin = False
for u in users:
    if u.role == Role.ADMIN:
        has_admin = True
        break
```

### `dataclasses` и `NamedTuple`

```python
from dataclasses import dataclass, field
from typing import NamedTuple


@dataclass
class UserDTO:
    """Объект передачи данных пользователя."""

    id: int
    name: str
    email: str
    is_active: bool = True
    tags: list[str] = field(default_factory=list)


class Coordinates(NamedTuple):
    """Неизменяемые координаты."""

    latitude: float
    longitude: float
```

### Контекстные менеджеры

**Всегда** используй `with` для файлов, соединений, транзакций:

```python
# ✅
with open('file.txt', 'r', encoding='utf-8') as f:
    content = f.read()

# Несколько менеджеров
with (
    open('src.txt', encoding='utf-8') as src,
    open('dst.txt', 'w', encoding='utf-8') as dst,
):
    dst.write(src.read())

# ❌ ЗАПРЕЩЕНО
f = open('file.txt')
content = f.read()
# f.close() может не выполниться при исключении
```

### `zip()`, `enumerate()`, `map()`

```python
# enumerate вместо ручного счётчика
for index, item in enumerate(items, start=1):
    print(f'{index}. {item}')

# zip для параллельного обхода
for user, score in zip(users, scores):
    ...

# map — только когда функция уже есть
clean_names = list(map(str.strip, raw_names))
# Но list comprehension читабельнее:
clean_names = [n.strip() for n in raw_names]
```

---

## Типизация (Type Hints)

Типизация обязательна для всех публичных функций и методов.
Исключение: когда типизация технически мешает
(например, callback API сторонних библиотек), допустим `Any`
или минимальные аннотации.

### Базовые аннотации

```python
# ✅ Полная типизация
def get_user(
    user_id: int,
    active_only: bool = True,
) -> User | None:
    ...


# Опциональные данные — X | None (не Optional[X])
def find_order(order_id: int) -> Order | None:
    ...
```

### Абстрактные контейнеры вместо конкретных

В параметрах функций указывай **минимально необходимый
интерфейс**, а не конкретную реализацию. Это делает функцию
гибче и помогает mypy ловить больше ошибок:

```python
from collections.abc import Iterable, Sequence, Mapping

# ❌ Привязка к конкретному типу
def process(items: list[str]) -> int:
    return sum(len(s) for s in items)

# ✅ Iterable — достаточно уметь итерироваться
def process(items: Iterable[str]) -> int:
    return sum(len(s) for s in items)

# ✅ Sequence — нужен доступ по индексу
def get_first(items: Sequence[str]) -> str:
    return items[0]

# ✅ Mapping — нужен доступ по ключу
def get_value(
    data: Mapping[str, int],
    key: str,
) -> int | None:
    return data.get(key)
```

Правило простое:

| Нужна возможность            | Используй       |
|------------------------------|------------------|
| Итерироваться (`for x in`)   | `Iterable[X]`   |
| Доступ по индексу (`xs[0]`)  | `Sequence[X]`   |
| Доступ по ключу (`d["k"]`)   | `Mapping[K, V]` |
| Мутировать коллекцию         | `list[X]` / `dict[K, V]` |

Конкретные типы (`list`, `dict`) используй:
в **возвращаемых** значениях, в полях **Pydantic-моделей**
и **dataclass**, а также когда функция **мутирует** коллекцию.

### TypedDict — типизированные словари

Если функция возвращает `dict` с известной структурой —
замени на `TypedDict`. Это позволяет mypy проверять ключи:

```python
from typing import TypedDict

# ❌ -> dict — неизвестно, какие ключи
def get_stats() -> dict:
    return {'users': 100, 'orders': 42}

# ✅ -> Stats — mypy знает структуру
class Stats(TypedDict):
    users: int
    orders: int

def get_stats() -> Stats:
    return {'users': 100, 'orders': 42}
```

Для необязательных ключей используй `NotRequired`
(Python 3.11+):

```python
from typing import NotRequired, TypedDict


class UserProfile(TypedDict):
    name: str
    email: str
    bio: NotRequired[str]
```

### Protocol — структурная типизация

`Protocol` описывает **интерфейс** без наследования.
Класс считается совместимым, если реализует нужные
методы — как в Go:

```python
from typing import Protocol


class Repository(Protocol):
    """Интерфейс репозитория — для DI и тестов."""

    async def get_by_id(
        self, entity_id: int,
    ) -> dict | None: ...

    async def save(self, entity: dict) -> None: ...


# UserRepository нигде не наследует Repository,
# но реализует его интерфейс — mypy это проверит.
class UserRepository:
    async def get_by_id(
        self, entity_id: int,
    ) -> dict | None:
        ...

    async def save(self, entity: dict) -> None:
        ...


class UserService:
    def __init__(self, repo: Repository) -> None:
        self._repo = repo
```

Плюсы: сервис зависит от интерфейса, не от реализации.
В тестах подставляй мок без наследования.

### Алиасы типов

```python
# Python 3.12+: оператор type
type UserId = int
type UserMap = dict[UserId, User]
type Handler = Callable[
    [Update, Context], Awaitable[None]
]

# Python < 3.12: TypeAlias
from typing import TypeAlias

UserId: TypeAlias = int
```

### Callable — типизация функций

```python
from collections.abc import Callable, Awaitable

# Синхронная функция: два int → int
type MathOp = Callable[[int, int], int]

def apply(op: MathOp, a: int, b: int) -> int:
    return op(a, b)

# Асинхронный хэндлер
type AsyncHandler = Callable[
    [Update, Context], Awaitable[None]
]
```

### Final, ClassVar

```python
from typing import Final, ClassVar


MAX_RETRIES: Final = 3  # нельзя переприсвоить

class Config:
    MAX_POOL: ClassVar[int] = 10  # атрибут класса
    name: str                     # атрибут экземпляра
```

### Чеклист типизации

- `Iterable` / `Sequence` / `Mapping` в параметрах,
  конкретные типы — в возвратах и полях моделей.
- `-> dict` → заменяй на `TypedDict` или `dataclass`.
- Зависимости между слоями — через `Protocol`.
- Сложные типы → алиас через `type` (Python 3.12).
- `Any` — только как крайняя мера с комментарием.

---

## Докстринги и комментарии

### Формат докстрингов

Используй **Google style**:

```python
def calculate_discount(
    price: Decimal,
    percent: Decimal,
    min_price: Decimal = Decimal('0'),
) -> Decimal:
    """Рассчитать цену с учётом скидки.

    Args:
        price: Исходная цена в рублях.
        percent: Процент скидки от 0 до 100.
        min_price: Минимальная допустимая цена.

    Returns:
        Цена после применения скидки.

    Raises:
        ValueError: Если percent не в диапазоне 0–100.

    Example:
        >>> calculate_discount(Decimal('1000'), Decimal('20'))
        Decimal('800')
    """
    if not 0 <= percent <= 100:
        raise ValueError(
            f'Скидка должна быть 0–100, получено: {percent}.'
        )
    discounted = price * (1 - percent / 100)
    return max(discounted, min_price)
```

### Правила комментариев

**Главное правило:** комментарий нужен там, где код
**правильный, но неочевидный**. Комментарий объясняет
**причину** (ПОЧЕМУ), а не действие (ЧТО).

Хороший код объясняет себя сам. Комментарий — это признание,
что код недостаточно выразителен. Если можно переименовать
переменную или выделить функцию — сделай это вместо
комментария.

#### ❌ Так делает ИИ — запрещено

```python
# Подключаемся к базе данных
db = Database(settings.database_url)

# Получаем пользователя по id
user = db.get_user(user_id)

# Проверяем, активен ли пользователь
if not user.is_active:
    # Возвращаем ошибку
    raise PermissionError('User is not active.')

# Возвращаем результат
return user
```

Это не документация — это пересказ кода другими словами.
Удали все эти комментарии: код станет лучше, не хуже.

#### ✅ Комментарий объясняет ПОЧЕМУ

```python
# Telegram не принимает сообщения длиннее 4096 символов.
# Обрезаем до 4093, чтобы добавить '...'
if len(text) > MAX_MESSAGE_LENGTH:
    text = text[:MAX_MESSAGE_LENGTH - 3] + '...'

# Argon2id намеренно медленный — это защита от брутфорса,
# а не баг производительности. Не «оптимизируй».
hashed = ph.hash(password)

# Грязный хак: библиотека X при None падает
# с AttributeError вместо ValueError — проверяем заранее
if value is None:
    return default

# TODO(anfi): заменить на WebSocket после апгрейда сервера
response = poll_for_result(task_id)
```

#### Где комментарий ТОЧНО нужен

- Нестандартное поведение внешней библиотеки или API
- Намеренное нарушение «правильного» подхода
  (workaround, legacy-совместимость)
- Сложный алгоритм: одна строка **перед блоком**
  с описанием идеи, не построчно
- `# TODO:` / `# FIXME:` с объяснением

#### Где комментарий ЗАПРЕЩЁН

- Перед каждой функцией вместо докстринга
- Перед любой очевидной операцией
- Закомментированный старый код — для этого есть git
- Комментарии-заголовки (`# === ИНИЦИАЛИЗАЦИЯ ===`) —
  для этого есть структура файла и докстринги

---
