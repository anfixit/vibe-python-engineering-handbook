## 25. Кеширование

### 25.1 Когда кешировать

Кеширование решает две проблемы: снижает нагрузку на БД
и ускоряет ответ. Но создаёт третью — инвалидация.

```
✅ Кешируй:
- Данные, которые читаются в 10+ раз чаще, чем пишутся
- Результаты тяжёлых запросов (агрегации, JOIN-ы)
- Ответы внешних API
- Статические справочники (города, категории)

❌ Не кешируй:
- Данные, которые меняются на каждый запрос
- Персональные данные пользователя (если кеш общий)
- Всё подряд «на всякий случай» — это усложняет отладку
```

### 25.2 In-memory vs Redis

```python
# --- In-memory (для одного процесса, dev, мелкий кеш) ---
# uv add cachetools
from cachetools import TTLCache

# Кеш на 1000 элементов, живёт 5 минут
_cache: TTLCache = TTLCache(
    maxsize=1000,
    ttl=CACHE_TTL_SHORT,
)


def get_cached_settings(key: str) -> dict | None:
    return _cache.get(key)


def set_cached_settings(key: str, value: dict) -> None:
    _cache[key] = value
```

```python
# --- Redis (для нескольких процессов, продакшн) ---
# uv add redis
import json
from redis.asyncio import Redis

redis = Redis.from_url(settings.redis_url)


async def get_cached(key: str) -> dict | None:
    data = await redis.get(key)
    if data is None:
        return None
    return json.loads(data)


async def set_cached(
    key: str,
    value: dict,
    ttl: int = CACHE_TTL_SHORT,
) -> None:
    await redis.set(key, json.dumps(value), ex=ttl)


async def invalidate(key: str) -> None:
    await redis.delete(key)


async def invalidate_pattern(pattern: str) -> None:
    """Удалить все ключи по паттерну (осторожно в проде)."""
    async for key in redis.scan_iter(match=pattern):
        await redis.delete(key)
```

Когда что:

| Критерий | In-memory | Redis |
|---|---|---|
| Один процесс | ✅ | Избыточно |
| Несколько воркеров | ❌ Рассинхрон | ✅ |
| Нужна персистентность | ❌ | ✅ |
| Скорость | ~нс | ~1 мс |
| Простота | ✅ | Нужен сервер |

### 25.3 Паттерн Cache-Aside

Самый распространённый паттерн: приложение управляет
кешем явно.

```python
async def get_user_profile(
    user_id: int,
) -> dict[str, Any]:
    """Cache-aside: сначала кеш, потом БД.

    Возвращает dict, а не модель — кеш хранит JSON.
    Для типизации можно использовать TypedDict.
    """
    cache_key = f'user:{user_id}:profile'

    # 1. Читаем из кеша
    cached = await get_cached(cache_key)
    if cached is not None:
        return cached

    # 2. Нет в кеше — идём в БД
    user = await repo.get_by_id(user_id)
    if user is None:
        raise NotFoundError('User', user_id)

    profile = UserProfile.model_validate(user).model_dump()

    # 3. Кладём в кеш
    await set_cached(cache_key, profile, ttl=CACHE_TTL_SHORT)

    return profile
```

### 25.4 Инвалидация

Самая сложная часть кеширования. Два подхода:

```python
# Подход 1: инвалидация при записи (точная)
async def update_user(
    user_id: int,
    data: UserUpdate,
) -> User:
    user = await repo.update(user_id, data)
    # Инвалидируем конкретный ключ
    await invalidate(f'user:{user_id}:profile')
    return user


# Подход 2: короткий TTL (ленивая)
# Ставим TTL = 60 секунд и не инвалидируем вручную.
# Данные обновятся сами через минуту.
# Подходит для некритичных данных (счётчики, статистика).
```

Правила:

- Предпочитай **явную инвалидацию** для данных,
  которые пользователь ожидает увидеть сразу после
  изменения (свой профиль, корзина, баланс).
- Предпочитай **короткий TTL** для данных, где задержка
  допустима (рейтинги, ленты, агрегации).
- **Никогда** не кешируй навечно (`ttl=0` или без TTL) —
  утечка памяти и несвежие данные гарантированы.

### 25.5 Cache stampede protection

Если TTL истёк и 1000 запросов одновременно идут в БД —
это cache stampede (thundering herd). Защита:

```python
import asyncio
from collections.abc import Awaitable, Callable
from cachetools import TTLCache

# Lock'и живут 5 минут, потом GC их уберёт.
# Без TTL — утечка памяти на долгоживущем сервисе.
_locks: TTLCache = TTLCache(maxsize=10_000, ttl=300)


async def get_with_lock(
    key: str,
    fetch_fn: Callable[[], Awaitable[dict]],
    ttl: int = CACHE_TTL_SHORT,
) -> dict:
    """Один запрос заполняет кеш, остальные ждут."""
    cached = await get_cached(key)
    if cached is not None:
        return cached

    if key not in _locks:
        _locks[key] = asyncio.Lock()

    async with _locks[key]:
        # Перепроверяем — может, уже заполнили
        cached = await get_cached(key)
        if cached is not None:
            return cached

        result = await fetch_fn()
        await set_cached(key, result, ttl=ttl)
        return result
```

---
