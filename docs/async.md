## Async: правила и ловушки

### Главное правило

**Никаких блокирующих вызовов в async-функциях.**
Один блокирующий вызов останавливает весь event loop —
все остальные запросы ждут.

```python
import asyncio
from pathlib import Path


# ❌ Блокирует event loop
async def read_config() -> str:
    return Path('config.txt').read_text()  # синхронный I/O


# ✅ Не блокирует
async def read_config() -> str:
    return await asyncio.to_thread(
        Path('config.txt').read_text
    )


# ❌ time.sleep блокирует всё
async def wait_and_do() -> None:
    import time
    time.sleep(5)  # ЗАПРЕЩЕНО


# ✅
async def wait_and_do() -> None:
    await asyncio.sleep(5)
```

### Частые блокирующие вызовы, о которых забывают

```python
# ❌ Всё это блокирует event loop:
import requests          # используй httpx
import psycopg2          # используй asyncpg или SQLAlchemy async
open('file').read()      # используй aiofiles или asyncio.to_thread
time.sleep(n)            # используй asyncio.sleep
subprocess.run(...)      # используй asyncio.create_subprocess_exec
```

### Параллельные задачи: `gather` vs последовательно

```python
import asyncio


# ❌ Последовательно — 3 секунды суммарно
async def fetch_all_slow() -> tuple:
    user = await get_user(user_id)      # 1 сек
    orders = await get_orders(user_id)  # 1 сек
    stats = await get_stats(user_id)    # 1 сек
    return user, orders, stats


# ✅ Параллельно — 1 секунда суммарно
async def fetch_all_fast() -> tuple:
    user, orders, stats = await asyncio.gather(
        get_user(user_id),
        get_orders(user_id),
        get_stats(user_id),
    )
    return user, orders, stats


# ✅ С обработкой частичных ошибок
results = await asyncio.gather(
    task_one(),
    task_two(),
    return_exceptions=True,  # не упадёт при ошибке одной задачи
)
for result in results:
    if isinstance(result, Exception):
        logger.error('Задача упала: %s', result)
```

### Таймауты — всегда

```python
# ❌ Может висеть вечно
result = await some_external_call()

# ✅ Явный таймаут
try:
    result = await asyncio.wait_for(
        some_external_call(),
        timeout=30.0,
    )
except asyncio.TimeoutError:
    logger.error('Таймаут внешнего вызова.')
    raise ServiceUnavailableError('Сервис не ответил.')
```

### Правильное создание и отмена задач

```python
# ❌ Задача может быть собрана GC до завершения
asyncio.create_task(background_job())

# ✅ Храни ссылку
background_tasks: set[asyncio.Task] = set()

task = asyncio.create_task(background_job())
background_tasks.add(task)
task.add_done_callback(background_tasks.discard)
```

### Async context managers для ресурсов

```python
# ✅ Соединения всегда через контекстный менеджер
async with aiofiles.open('file.txt', encoding='utf-8') as f:
    content = await f.read()

async with httpx.AsyncClient() as client:
    response = await client.get(url)

# Для долгоживущего клиента — инициализация при старте
# приложения (см. раздел lifespan в FastAPI)
```

### Graceful shutdown

Для long-running сервисов (боты, воркеры) важно
корректно завершать работу — дождаться текущих задач,
закрыть соединения, не потерять данные:

```python
import asyncio
import signal
import logging

logger = logging.getLogger(__name__)


class GracefulShutdown:
    """Обработчик корректного завершения."""

    def __init__(self) -> None:
        self._shutdown_event = asyncio.Event()

    def setup(self, loop: asyncio.AbstractEventLoop) -> None:
        """Зарегистрировать обработчики сигналов."""
        for sig in (signal.SIGINT, signal.SIGTERM):
            loop.add_signal_handler(
                sig,
                self._handle_signal,
                sig,
            )

    def _handle_signal(self, sig: signal.Signals) -> None:
        logger.info(
            'Получен сигнал %s, завершаемся...',
            sig.name,
        )
        self._shutdown_event.set()

    async def wait(self) -> None:
        """Ожидать сигнала завершения."""
        await self._shutdown_event.wait()


# Использование в боте
async def main() -> None:
    bot = Bot(token=settings.bot_token.get_secret_value())
    dp = Dispatcher(storage=MemoryStorage())

    shutdown = GracefulShutdown()
    shutdown.setup(asyncio.get_running_loop())

    polling_task = asyncio.create_task(
        dp.start_polling(bot)
    )

    await shutdown.wait()

    # Корректно завершаем
    await dp.stop_polling()
    await bot.session.close()
    logger.info('Бот остановлен корректно.')
```

---

## Транзакции и миграции (Alembic)

### Транзакции — явно, всегда

Правило: **всё, что меняет несколько записей, — в транзакции.**
Если одна операция упала — откатываются все.

```python
# FastAPI + SQLAlchemy async
from decimal import Decimal
from sqlalchemy.ext.asyncio import AsyncSession


async def transfer_funds(
    session: AsyncSession,
    from_id: int,
    to_id: int,
    amount: Decimal,
) -> None:
    async with session.begin():
        sender = await session.get(Account, from_id)
        receiver = await session.get(Account, to_id)

        if sender.balance < amount:
            raise InsufficientFundsError(
                f'Недостаточно средств: {sender.balance}.'
            )

        sender.balance -= amount
        receiver.balance += amount
        # session.begin() сам вызовет commit() или rollback()


# Django — декоратор для метода сервиса
from django.db import transaction


@transaction.atomic
def transfer_funds(from_id, to_id, amount):
    ...
```

**Внимание:** для денежных операций используй `Decimal`,
а не `float`. `0.1 + 0.2 != 0.3` с float.
Подробнее — см. раздел 18.6 «Decimal для денег».

### Alembic: настройка

```bash
uv add alembic sqlalchemy asyncpg
alembic init -t async migrations
```

```python
# migrations/env.py — подключи свои модели
from myapp.models import Base  # импортируй все модели
target_metadata = Base.metadata
```

### Workflow миграций

```bash
# Создать миграцию автоматически (после изменения модели)
alembic revision --autogenerate -m "add users table"

# Всегда проверяй сгенерированный файл перед применением!
# Alembic не всегда корректно определяет изменения.

# Применить все миграции
alembic upgrade head

# Откатить последнюю
alembic downgrade -1

# Текущее состояние
alembic current

# История
alembic history --verbose
```

### Правила миграций

```
✅ Миграции только вперёд в продакшне.
✅ Каждая миграция — атомарная операция.
✅ Никогда не редактируй уже применённую миграцию.
✅ Тестируй upgrade И downgrade локально.
✅ Добавление колонки — всегда с default или nullable.
❌ Никогда не удаляй колонку в той же миграции,
   где удаляешь код который её использует.
   Сначала деплой без кода → потом миграция удаления.
```

Пример безопасного удаления колонки:

```
Шаг 1: убрать использование колонки в коде → деплой
Шаг 2: alembic revision --autogenerate -m "drop old_col"
Шаг 3: alembic upgrade head
```

### Database connection pooling

Пул соединений для SQLAlchemy async — настраивай явно:

```python
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    async_sessionmaker,
)


engine = create_async_engine(
    str(settings.database_url),
    pool_size=20,           # постоянные соединения
    max_overflow=10,        # дополнительные при пиках
    pool_recycle=3600,      # пересоздавать через 1 час
    pool_pre_ping=True,     # проверять живость соединения
    pool_timeout=30,        # ждать свободное соединение
    echo=settings.debug,    # логировать SQL в dev
)

session_factory = async_sessionmaker(
    engine,
    expire_on_commit=False,
)
```

Правила:

- `pool_pre_ping=True` — **всегда**, иначе после
  перезагрузки БД получишь мёртвые соединения.
- `pool_recycle` — ставь меньше, чем `wait_timeout`
  в PostgreSQL/MySQL.
- `echo=True` — **только в dev**, в продакшне выключай.

### Decimal для денег

`float` не подходит для финансовых вычислений.
`0.1 + 0.2 == 0.30000000000000004` — это не баг Python,
это свойство IEEE 754.

```python
from decimal import Decimal

# ❌ ЗАПРЕЩЕНО для денег
price = 19.99                     # float
total = price * 3                 # 59.96999...
discount = price * 0.1            # 1.9990000...

# ✅ Decimal для денег
price = Decimal('19.99')
total = price * 3                 # Decimal('59.97')
discount = price * Decimal('0.1') # Decimal('1.999')

# ✅ Округление
total = (price * quantity).quantize(Decimal('0.01'))
```

В моделях SQLAlchemy:

```python
from sqlalchemy import Numeric
from sqlalchemy.orm import Mapped, mapped_column


class Product(Base):
    __tablename__ = 'products'

    price: Mapped[Decimal] = mapped_column(
        Numeric(precision=10, scale=2),
    )
```

В Pydantic-схемах:

```python
from decimal import Decimal
from pydantic import BaseModel, Field


class ProductCreate(BaseModel):
    price: Decimal = Field(
        gt=0,
        decimal_places=2,
        description='Цена в рублях',
    )
```

Правило: для денег — **только `Decimal`**, никогда `float`.
Создавай `Decimal` из строк (`Decimal('19.99')`),
не из float (`Decimal(19.99)` — унаследует неточность).

### N+1 в SQLAlchemy

Django имеет `select_related` / `prefetch_related`.
В SQLAlchemy async используй `selectinload` / `joinedload`:

```python
from sqlalchemy.orm import selectinload, joinedload
from sqlalchemy import select

# ❌ N+1: для каждого поста — отдельный запрос автора
posts = await session.execute(select(Post))
for post in posts.scalars():
    print(post.author.name)  # +1 запрос на каждый пост

# ✅ joinedload: JOIN в одном запросе (для to-one)
posts = await session.execute(
    select(Post).options(joinedload(Post.author))
)

# ✅ selectinload: SELECT IN (для to-many)
users = await session.execute(
    select(User).options(selectinload(User.posts))
)
```

---
