## Aiogram (Telegram-боты) специфика

```python
# bot.py
import asyncio
import logging

from aiogram import Bot, Dispatcher
from aiogram.fsm.storage.memory import MemoryStorage

from myapp.config import settings
from myapp.handlers import router

logger = logging.getLogger(__name__)


async def main() -> None:
    """Точка входа бота."""
    validate_settings()

    bot = Bot(token=settings.bot_token.get_secret_value())
    # MemoryStorage — данные FSM теряются при перезапуске.
    # Для прода: uv add aiogram[redis]
    # storage = RedisStorage.from_url(settings.redis_url)
    dp = Dispatcher(storage=MemoryStorage())
    dp.include_router(router)

    logger.info('Бот запущен.')
    try:
        await dp.start_polling(bot)
    finally:
        await bot.session.close()
        logger.info('Бот остановлен.')


if __name__ == '__main__':
    asyncio.run(main())
```

Секрет: `bot_token` достаётся через `.get_secret_value()` —
в логах не засветится.

---

## Telegram-боты (Aiogram 3)

### Структура проекта

```
bot/
├── handlers/
│   ├── __init__.py
│   ├── common.py       # /start, /help
│   ├── registration.py # FSM регистрации
│   └── admin.py
├── keyboards/
│   ├── inline.py
│   └── reply.py
├── middlewares/
│   ├── throttling.py
│   └── db_session.py
├── services/
│   └── user_service.py
├── states/
│   └── registration.py
├── filters/
│   └── admin_filter.py
├── config.py
├── constants.py
└── main.py
```

### FSM: конечные автоматы состояний

```python
# states/registration.py
from aiogram.fsm.state import State, StatesGroup


class Registration(StatesGroup):
    waiting_for_name = State()
    waiting_for_phone = State()
    waiting_for_confirm = State()
```

```python
# handlers/registration.py
from aiogram import Router, F
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.types import Message
from myapp.states.registration import Registration

router = Router()


@router.message(Command('register'))
async def start_registration(
    message: Message,
    state: FSMContext,
) -> None:
    await state.set_state(Registration.waiting_for_name)
    await message.answer('Введите ваше имя:')


@router.message(Registration.waiting_for_name)
async def process_name(
    message: Message,
    state: FSMContext,
) -> None:
    if not message.text:
        await message.answer('Введите имя текстом.')
        return

    await state.update_data(name=message.text)
    await state.set_state(Registration.waiting_for_phone)
    await message.answer('Введите номер телефона:')


@router.message(Registration.waiting_for_phone)
async def process_phone(
    message: Message,
    state: FSMContext,
) -> None:
    data = await state.get_data()
    name = data['name']
    # сохранение в БД через сервис
    await state.clear()
    await message.answer(
        f'Готово, {name}! Регистрация завершена.'
    )
```

### Middleware: БД-сессия и троттлинг

```python
# middlewares/db_session.py
from typing import Any, Callable, Awaitable
from aiogram import BaseMiddleware
from aiogram.types import TelegramObject
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker


class DbSessionMiddleware(BaseMiddleware):
    """Инжектировать сессию БД в каждый хэндлер."""

    def __init__(
        self,
        session_factory: async_sessionmaker,
    ) -> None:
        self._factory = session_factory

    async def __call__(
        self,
        handler: Callable[
            [TelegramObject, dict[str, Any]],
            Awaitable[Any],
        ],
        event: TelegramObject,
        data: dict[str, Any],
    ) -> Any:
        async with self._factory() as session:
            data['session'] = session
            return await handler(event, data)
```

```python
# middlewares/throttling.py
import asyncio
from cachetools import TTLCache
from aiogram import BaseMiddleware
from aiogram.types import Message

# Не более 1 сообщения в секунду на пользователя
THROTTLE_CACHE: TTLCache = TTLCache(
    maxsize=10_000, ttl=1
)


class ThrottlingMiddleware(BaseMiddleware):

    async def __call__(self, handler, event, data):
        if isinstance(event, Message):
            user_id = event.from_user.id
            if user_id in THROTTLE_CACHE:
                return  # игнорируем
            THROTTLE_CACHE[user_id] = True
        return await handler(event, data)
```

### Клавиатуры — фабрики, не хардкод

```python
# keyboards/inline.py
from aiogram.types import InlineKeyboardMarkup
from aiogram.utils.keyboard import InlineKeyboardBuilder


def confirm_keyboard() -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    builder.button(text='✅ Подтвердить', callback_data='confirm')
    builder.button(text='❌ Отменить', callback_data='cancel')
    builder.adjust(2)
    return builder.as_markup()


def pagination_keyboard(
    page: int,
    total_pages: int,
) -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    if page > 1:
        builder.button(
            text='◀️',
            callback_data=f'page:{page - 1}',
        )
    builder.button(
        text=f'{page}/{total_pages}',
        callback_data='noop',
    )
    if page < total_pages:
        builder.button(
            text='▶️',
            callback_data=f'page:{page + 1}',
        )
    return builder.as_markup()
```

### Антипаттерны в ботах

```python
# ❌ Логика в хэндлере
@router.message(Command('stats'))
async def show_stats(message: Message) -> None:
    # 50 строк SQL и бизнес-логики прямо здесь — ЗАПРЕЩЕНО
    users = await session.execute(
        select(User).where(...)
    )
    ...

# ✅ Хэндлер только оркестрирует
@router.message(Command('stats'))
async def show_stats(
    message: Message,
    service: StatsService,
) -> None:
    stats = await service.get_summary()
    await message.answer(format_stats(stats))


# ❌ Глобальный bot-объект
bot = Bot(token=TOKEN)  # в модуле на верхнем уровне


# ✅ Через DI или lifespan
# bot передаётся через data['bot'] или через middleware
```

---
