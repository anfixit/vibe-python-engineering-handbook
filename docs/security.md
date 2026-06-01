## 3. Секреты и конфигурация

### 3.1 Иерархия надёжности (от лучшего к худшему)

```
1. Менеджер секретов (Vault, AWS Secrets Manager) ← ЛУЧШИЙ
2. Переменные окружения ОС (export VAR=value)
3. .env файл + pydantic-settings               ← ДЛЯ РАЗРАБОТКИ
4. config.py с os.environ.get()                ← ПРИЕМЛЕМО
5. Захардкоженные значения                     ← ЗАПРЕЩЕНО
```

### 3.2 Правильная работа с `.env`

**`.env` — никогда не попадает в git.**

`.gitignore` (обязательно):

```
.env
.env.local
.env.production
*.pem
*.key
```

**`.env.example`** — шаблон без реальных значений (в git):

```bash
# .env.example
BOT_TOKEN=your_telegram_bot_token_here
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
SECRET_KEY=generate_with_openssl_rand_hex_32
DEBUG=false
ALLOWED_HOSTS=localhost,127.0.0.1
SENTRY_DSN=
REDIS_URL=redis://localhost:6379/0
EXTERNAL_API_URL=
```

**Никогда не делай так:**

```python
# ❌ ЗАПРЕЩЕНО
BOT_TOKEN = '1234567890:AABBcc...'
DATABASE_URL = 'postgresql://admin:password@prod-server/db'
```

### 3.3 Pydantic Settings (рекомендованный способ)

Устанавливай: `uv add pydantic-settings`

```python
# config.py
from pydantic import Field, PostgresDsn, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """Конфигурация приложения из env-переменных."""

    model_config = SettingsConfigDict(
        env_file='.env',
        env_file_encoding='utf-8',
        case_sensitive=False,
    )

    # Telegram
    bot_token: SecretStr = Field(
        description='Токен Telegram-бота'
    )

    # База данных — используй ИЛИ database_url (FastAPI),
    # ИЛИ отдельные db_* поля (Django). Не оба сразу.
    database_url: PostgresDsn | None = Field(
        default=None,
        description='URL подключения к PostgreSQL',
    )

    # Приложение
    debug: bool = Field(default=False)
    secret_key: SecretStr = Field(
        min_length=32,
        description='Секретный ключ приложения',
    )
    allowed_hosts: list[str] = Field(
        default=['localhost', '127.0.0.1']
    )

    # База данных (альтернатива database_url для Django)
    db_name: str = Field(default='')
    db_user: str = Field(default='')
    db_password: SecretStr = Field(default='')
    db_host: str = Field(default='localhost')
    db_port: int = Field(default=5432)

    # Безопасность
    argon2_time_cost: int = Field(default=2)
    argon2_memory_cost: int = Field(default=65536)
    rate_limit_per_minute: int = Field(default=60)

    # Мониторинг
    sentry_dsn: SecretStr | None = Field(
        default=None,
        description='Sentry DSN (содержит auth-ключ)',
    )

    # Внешние сервисы (опциональные)
    redis_url: str = Field(
        default='redis://localhost:6379/0',
        description='URL подключения к Redis',
    )
    external_api_url: str = Field(
        default='',
        description='Base URL внешнего API',
    )


# Синглтон — создаётся один раз.
# Если обязательные поля не заданы, Pydantic сам
# выбросит ValidationError с понятным traceback —
# дополнительная проверка не нужна.
settings = Settings()
```

**Использование:**

```python
# В любом модуле
from myapp.config import settings

token = settings.bot_token.get_secret_value()
# FastAPI — через database_url:
db_url = str(settings.database_url)
# Django — через отдельные поля (см. 10.3)
```

**Почему `SecretStr`:** значение не попадает в логи.
При `print(settings.bot_token)` выведется `**********`.

### 3.4 Проверка наличия переменных при старте

Pydantic Settings сам выбрасывает `ValidationError`, если
обязательные поля (без `default`) не заполнены, а также
проверяет `min_length`, `pattern` и другие ограничения
из `Field`. Поэтому ручная проверка нужна **только для
бизнес-правил**, которые Pydantic не покрывает:

```python
# main.py или __init__.py
def validate_settings() -> None:
    """Дополнительные проверки сверх Pydantic-валидации."""
    token = settings.bot_token.get_secret_value()
    # Проверяем формат токена Telegram: 123456789:AABBcc...
    parts = token.split(':')
    if len(parts) != 2 or not parts[0].isdigit():
        raise ValueError(
            'BOT_TOKEN не похож на Telegram-токен. '
            'Формат: 123456789:AABBcc...'
        )
```

---

## 7. Обработка ошибок и логирование

### 7.1 Исключения

```python
# ✅ Конкретные исключения
try:
    result = int(user_input)
except ValueError as exc:
    logger.warning('Неверный формат числа: %s', user_input)
    raise ValidationError(f'Ожидалось число, получено: '
                          f'{user_input!r}') from exc

# ❌ Голый except — ЗАПРЕЩЁН
try:
    ...
except:  # Поймает даже KeyboardInterrupt!
    pass

# ❌ Слишком широкий except
try:
    ...
except Exception:
    pass
```

Только строки, которые **могут** вызвать исключение,
помещаются в `try`. Не весь блок кода.

### 7.2 Кастомные исключения

```python
# exceptions.py
class AppError(Exception):
    """Базовое исключение приложения."""


class ValidationError(AppError):
    """Ошибка валидации входных данных."""


class NotFoundError(AppError):
    """Запрошенный ресурс не найден."""

    def __init__(self, resource: str, resource_id: int) -> None:
        self.resource = resource
        self.resource_id = resource_id
        super().__init__(
            f'{resource} с id={resource_id} не найден.'
        )


class ServiceUnavailableError(AppError):
    """Внешний сервис недоступен."""


class ExternalServiceError(AppError):
    """Ошибка внешнего сервиса."""


class AlreadyExistsError(AppError):
    """Ресурс уже существует."""


class InvalidCredentialsError(AppError):
    """Неверные учётные данные."""


class InsufficientFundsError(AppError):
    """Недостаточно средств."""


class InsufficientStockError(AppError):
    """Недостаточно товара на складе."""
```

### 7.3 Логирование

```python
# logger.py
import logging
import sys
from pathlib import Path

LOG_FORMAT = (
    '%(asctime)s | %(levelname)-8s | '
    '%(name)s:%(lineno)d | %(message)s'
)
LOG_DIR = Path(__file__).parent.parent / 'logs'


def setup_logger(name: str) -> logging.Logger:
    """Настроить логгер с выводом в файл и консоль."""
    LOG_DIR.mkdir(exist_ok=True)

    logger = logging.getLogger(name)
    logger.setLevel(logging.DEBUG)

    # Не добавляем handlers повторно при повторном вызове.
    if logger.handlers:
        return logger

    logger.propagate = False

    # Консоль
    console = logging.StreamHandler(sys.stdout)
    console.setLevel(logging.INFO)
    console.setFormatter(logging.Formatter(LOG_FORMAT))

    # Файл
    file_handler = logging.FileHandler(
        LOG_DIR / f'{name}.log',
        encoding='utf-8',
    )
    file_handler.setLevel(logging.DEBUG)
    file_handler.setFormatter(logging.Formatter(LOG_FORMAT))

    logger.addHandler(console)
    logger.addHandler(file_handler)
    return logger


logger = setup_logger('myapp')
```

**Правила логирования:**

```python
# ✅ Используй %s, не f-строки — ленивое форматирование
logger.info('Пользователь %s вошёл в систему', user.id)
logger.error('Ошибка запроса к API: %s %s', resp.status, url)

# ❌ ЗАПРЕЩЕНО — чувствительные данные в логах
logger.info('Пароль: %s', password)
logger.debug('Токен: %s', token)
logger.info('Карта: %s', card_number)

# ✅ Используй exc_info для трассировки
try:
    ...
except SomeError:
    logger.exception('Описание того, что происходило')
    # logger.exception автоматически добавляет traceback
```

### 7.4 Structured logging для продакшна

Раздел 7.3 выше — достаточен для скриптов, CLI-утилит
и простых ботов. Но для веб-приложений и API в продакшне,
где логи уходят в агрегаторы (ELK, Loki, Datadog,
CloudWatch), нужен JSON-формат. Используй `structlog`:

```python
# uv add structlog
import logging
import structlog


def setup_structlog(json_output: bool = False) -> None:
    """Настроить structlog: текст для dev, JSON для prod."""
    processors = [
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt='iso'),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
    ]

    if json_output:
        processors.append(
            structlog.processors.JSONRenderer()
        )
    else:
        processors.append(
            structlog.dev.ConsoleRenderer()
        )

    structlog.configure(
        processors=processors,
        wrapper_class=structlog.make_filtering_bound_logger(
            logging.INFO
        ),
    )


# Использование
logger = structlog.get_logger()

# ✅ Структурированные поля вместо конкатенации
logger.info(
    'user_login',
    user_id=42,
    ip='192.168.1.1',
    method='email',
)
# JSON: {"event":"user_login","user_id":42,"ip":"192.168.1.1",...}

# ✅ Bind для контекста на весь запрос
log = logger.bind(request_id='abc-123', user_id=42)
log.info('processing_started')
log.info('processing_completed', duration_ms=150)
```

Правило: в продакшне **всегда JSON**, локально — текст.
Переключай через переменную окружения.
Вызывай настройку логирования после инициализации `settings`:

```python
setup_structlog(json_output=not settings.debug)
```

---

## 8. Безопасность

### 8.1 Пароли: Argon2id

```python
# uv add argon2-cffi
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

ph = PasswordHasher(
    time_cost=2,       # итерации
    memory_cost=65536, # 64 MB
    parallelism=2,
)


def hash_password(plain: str) -> str:
    """Захэшировать пароль с Argon2id."""
    return ph.hash(plain)


def verify_password(hashed: str, plain: str) -> bool:
    """Проверить пароль. False при несовпадении."""
    try:
        return ph.verify(hashed, plain)
    except VerifyMismatchError:
        return False
```

**Никогда:** MD5, SHA1, SHA256 для паролей. Только Argon2id,
bcrypt или scrypt.

### 8.2 Валидация входных данных (Pydantic v2)

```python
from pydantic import BaseModel, EmailStr, Field, field_validator


class UserCreate(BaseModel):
    """Схема создания пользователя."""

    username: str = Field(
        min_length=3,
        max_length=MAX_USERNAME_LENGTH,
        pattern=r'^[a-zA-Z0-9_]+$',
    )
    email: EmailStr
    password: str = Field(min_length=MIN_PASSWORD_LENGTH)

    @field_validator('password')
    @classmethod
    def password_strength(cls, v: str) -> str:
        """Проверить сложность пароля."""
        if not any(c.isupper() for c in v):
            raise ValueError(
                'Пароль должен содержать заглавную букву.'
            )
        return v
```

**Внимание:** `@validator` — это Pydantic v1 API,
deprecated в v2. Используй `@field_validator`.

### 8.3 Защита API

```python
from datetime import datetime, timedelta, timezone

# Никогда не передавай в JWT чувствительные данные
# ✅ Только id и роль
payload = {
    'sub': str(user.id),
    'role': user.role,
    'exp': datetime.now(timezone.utc) + timedelta(hours=1),
}

# ❌ ЗАПРЕЩЕНО
payload = {
    'email': user.email,    # лишнее
    'password': ...,        # преступление
    'card_number': ...,     # преступление
}
```

**Внимание:** `datetime.utcnow()` возвращает naive datetime
(без timezone) и потому не рекомендуется.
В новых проектах используй `datetime.now(timezone.utc)`.
См. раздел 8.12 «Datetime: всегда timezone-aware».

### 8.4 Rate limiting (FastAPI пример)

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)


@router.post('/auth/login')
@limiter.limit('5/minute')
async def login(request: Request, data: LoginSchema):
    ...
```

Rate limiting нужен **не только на логин** — на все
auth-эндпоинты: регистрацию, сброс пароля, подтверждение
email, refresh токена.

### 8.5 CORS

CORS настраивается неправильно чаще, чем любой другой
заголовок. Главная ошибка — `allow_origins=["*"]`
в продакшне.

**FastAPI:**

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    # ❌ НИКОГДА в продакшне:
    # allow_origins=['*'],

    # ✅ Только свои домены:
    allow_origins=[
        'https://myapp.com',
        'https://www.myapp.com',
    ],
    allow_credentials=True,   # только если нужны куки/auth
    allow_methods=['GET', 'POST', 'PUT', 'DELETE'],
    allow_headers=['Authorization', 'Content-Type'],
)
```

**Django (`django-cors-headers`):**

```python
# settings.py
INSTALLED_APPS = [
    ...
    'corsheaders',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',  # ПЕРВЫМ в списке
    'django.middleware.common.CommonMiddleware',
    ...
]

# ❌ ЗАПРЕЩЕНО в продакшне
# CORS_ALLOW_ALL_ORIGINS = True

# ✅
CORS_ALLOWED_ORIGINS = [
    'https://myapp.com',
    'https://www.myapp.com',
]
CORS_ALLOW_CREDENTIALS = True
```

Правило: `allow_origins=["*"]` допустимо только для
**публичных read-only API** без авторизации.
Если есть куки или `Authorization` — только явные домены.

### 8.6 CSP (Content Security Policy)

CSP-заголовки — это второй рубеж защиты от XSS.
Браузер не выполнит скрипт, не разрешённый политикой,
даже если атака проникла в HTML.

**FastAPI — через middleware:**

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request


class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """Заголовки безопасности ко всем ответам."""

    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        response.headers['Content-Security-Policy'] = (
            "default-src 'self'; "
            "script-src 'self'; "
            "style-src 'self' 'unsafe-inline'; "
            "img-src 'self' data: https:; "
            "font-src 'self'; "
            "connect-src 'self'; "
            "frame-ancestors 'none';"
        )
        response.headers['X-Content-Type-Options'] = (
            'nosniff'
        )
        response.headers['X-Frame-Options'] = 'DENY'
        response.headers['Referrer-Policy'] = (
            'strict-origin-when-cross-origin'
        )
        response.headers[
            'Strict-Transport-Security'
        ] = 'max-age=31536000; includeSubDomains'
        return response


app.add_middleware(SecurityHeadersMiddleware)
```

**Django (`django-csp`):**

```python
# uv add django-csp
# settings.py
MIDDLEWARE = [
    ...
    'csp.middleware.CSPMiddleware',
]

CSP_DEFAULT_SRC = ("'self'",)
CSP_SCRIPT_SRC = ("'self'",)
CSP_STYLE_SRC = ("'self'", "'unsafe-inline'")
CSP_IMG_SRC = ("'self'", 'data:', 'https:')
CSP_FRAME_ANCESTORS = ("'none'",)
```

Минимальный набор заголовков для любого приложения:

```
Content-Security-Policy   ← защита от XSS
X-Content-Type-Options    ← защита от MIME-sniffing
X-Frame-Options           ← защита от clickjacking
Strict-Transport-Security ← принудительный HTTPS
Referrer-Policy           ← контроль реферера
```

### 8.7 Безопасные куки

Если используешь сессии или refresh-токены в куках:

```python
# FastAPI
from fastapi import Response


def set_auth_cookie(response: Response, token: str) -> None:
    response.set_cookie(
        key='refresh_token',
        value=token,
        httponly=True,   # JS не может прочитать — защита от XSS
        secure=True,     # только по HTTPS
        samesite='lax',  # защита от CSRF
        max_age=TOKEN_EXPIRY,
        path='/api/auth',  # только нужный путь
    )
```

```python
# Django
SESSION_COOKIE_HTTPONLY = True   # всегда True
SESSION_COOKIE_SECURE = True     # True в продакшне
SESSION_COOKIE_SAMESITE = 'Lax'
CSRF_COOKIE_HTTPONLY = True
CSRF_COOKIE_SECURE = True
```

### 8.8 RLS (Row-Level Security) — PostgreSQL / Supabase

RLS — это защита на уровне БД: пользователь физически
не может получить чужие строки, даже если в коде ошибка.

**Supabase (включается в настройках таблицы):**

```sql
-- Включить RLS на таблице
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

-- Политика: пользователь видит только свои записи
CREATE POLICY "Users see own posts"
ON posts FOR SELECT
USING (auth.uid() = user_id);

-- Политика: пользователь редактирует только свои
CREATE POLICY "Users update own posts"
ON posts FOR UPDATE
USING (auth.uid() = user_id);

-- Политика: пользователь удаляет только свои
CREATE POLICY "Users delete own posts"
ON posts FOR DELETE
USING (auth.uid() = user_id);
```

**PostgreSQL (без Supabase):**

```sql
-- Создать роль для приложения
CREATE ROLE app_user;

-- Передавать user_id через сессионную переменную
-- В приложении перед каждым запросом:
-- SET app.current_user_id = '123';

CREATE POLICY "tenant_isolation"
ON orders FOR ALL
USING (
    user_id = current_setting('app.current_user_id')::int
);
```

**Важно:** RLS — дополнительный рубеж, а не замена
проверок в коде. Проверяй права и в сервисном слое,
и на уровне БД.

### 8.9 Аудит зависимостей

```bash
# Python — проверка CVE в зависимостях
uv run pip-audit

# Или через uv
uv run pip-audit

# Автоматически в GitHub Actions
- name: Audit dependencies
  run: uv run pip-audit --strict
```

Добавь в CI/CD — пусть падает при критических уязвимостях.

### 8.10 SQL-инъекции: абсолютный запрет на конкатенацию

ORM защищает от инъекций автоматически. Но когда
ORM не хватает и ты пишешь raw SQL — **никогда**
не подставляй значения через f-строки или конкатенацию.

```python
# ❌ ЗАПРЕЩЕНО — SQL-инъекция
query = f"SELECT * FROM users WHERE email = '{email}'"
cursor.execute(
    "SELECT * FROM users WHERE id = " + str(user_id)
)

# ❌ ЗАПРЕЩЕНО — даже через format
query = "DELETE FROM orders WHERE id = {}".format(order_id)

# ✅ Параметризованный запрос (SQLAlchemy)
from sqlalchemy import text

result = await session.execute(
    text('SELECT * FROM users WHERE email = :email'),
    {'email': email},
)

# ✅ Параметризованный запрос (psycopg / asyncpg)
await cursor.execute(
    'SELECT * FROM users WHERE id = $1',
    (user_id,),
)

# ✅ Django ORM raw-запрос
User.objects.raw(
    'SELECT * FROM users WHERE email = %s',
    [email],
)
```

Правило: **любое значение из внешнего источника**
(пользователь, API, файл) идёт **только** через
параметры запроса, никогда через конкатенацию.

### 8.11 Безопасность загрузки файлов

Если приложение принимает файлы от пользователей:

```python
import secrets
from pathlib import Path

import magic  # uv add python-magic


# --- Константы ---
MAX_UPLOAD_SIZE_BYTES = 10 * 1024 * 1024  # 10 MB
ALLOWED_MIME_TYPES = frozenset({
    'image/jpeg',
    'image/png',
    'image/webp',
    'application/pdf',
})
UPLOAD_DIR = Path('/var/uploads')  # вне webroot!


def validate_upload(
    content: bytes,
    original_filename: str,
) -> str:
    """Валидировать файл и вернуть безопасное имя.

    Raises:
        ValidationError: Если файл не прошёл проверку.
    """
    # 1. Размер
    if len(content) > MAX_UPLOAD_SIZE_BYTES:
        raise ValidationError(
            f'Файл слишком большой: '
            f'максимум {MAX_UPLOAD_SIZE_BYTES // 1024 // 1024} MB.'
        )

    # 2. MIME-type по содержимому, НЕ по расширению
    mime = magic.from_buffer(content, mime=True)
    if mime not in ALLOWED_MIME_TYPES:
        raise ValidationError(
            f'Тип файла {mime!r} не разрешён.'
        )

    # 3. Генерируем случайное имя (не доверяем оригиналу)
    ext = Path(original_filename).suffix.lower()
    safe_name = f'{secrets.token_hex(16)}{ext}'

    return safe_name


# ❌ ЗАПРЕЩЕНО
# 1. Доверять расширению файла
# 2. Сохранять оригинальное имя (path traversal: ../../etc/passwd)
# 3. Хранить загрузки в static/media, доступном напрямую
# 4. Не проверять MIME-type по содержимому
```

### 8.12 Datetime: всегда timezone-aware

Naive datetime (без timezone) — источник багов
при работе с БД, API и разными серверами.

```python
from datetime import datetime, timezone, timedelta

# ❌ ЗАПРЕЩЕНО — naive datetime
now = datetime.now()           # нет timezone
now = datetime.utcnow()       # naive, не рекомендуется

# ✅ Всегда aware
now = datetime.now(timezone.utc)
expires = now + timedelta(hours=1)

# ✅ Для хранения в БД — всегда UTC
created_at = datetime.now(timezone.utc)

# ✅ Для отображения — конвертируй в нужную TZ
from zoneinfo import ZoneInfo

msk = ZoneInfo('Europe/Moscow')
local_time = created_at.astimezone(msk)
```

Правило: **всё хранится в UTC**, конвертация в локальное
время — только в слое отображения (шаблоны, ответ API).

### 8.13 Защита от mass-assignment

Никогда не передавай данные пользователя напрямую
в модель БД без фильтрации полей:

```python
# ❌ ЗАПРЕЩЕНО — пользователь может передать role='admin'
user = User(**request.dict())
User.objects.create(**request.data)

# ✅ Явное перечисление полей
user = User(
    email=data.email,
    name=data.name,
    hashed_password=hash_password(data.password),
    # role НЕ берём из запроса
)

# ✅ Pydantic-схема как фильтр
class UserCreate(BaseModel):
    """Только поля, которые может задать пользователь."""
    email: EmailStr
    name: str
    password: str
    # role — нет в схеме, значит не попадёт из запроса
```

---
