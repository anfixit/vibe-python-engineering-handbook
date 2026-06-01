## Django / DRF специфика

### Модели

```python
from django.db import models


class TimestampMixin(models.Model):
    """Абстрактная модель с временны́ми метками."""

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class Post(TimestampMixin):
    """Публикация."""

    title = models.CharField(max_length=MAX_TITLE_LENGTH)
    content = models.TextField()
    author = models.ForeignKey(
        'auth.User',
        on_delete=models.CASCADE,
        related_name='posts',
    )

    class Meta:
        ordering = ('-created_at',)
        verbose_name = 'Публикация'
        verbose_name_plural = 'Публикации'

    def __str__(self) -> str:
        return self.title[:20]
```

### Оптимизация запросов

```python
# ✅ Один запрос вместо N+1
posts = Post.objects.select_related(
    'author'
).prefetch_related(
    'tags',
    'comments',
).filter(is_published=True)

# Массовое создание
Post.objects.bulk_create([
    Post(title=t, author=user)
    for t in titles
])

# Список значений одного поля
user_ids = User.objects.values_list('id', flat=True)
```

### Секреты Django

```python
# settings.py
from myapp.config import settings

SECRET_KEY = settings.secret_key.get_secret_value()
DEBUG = settings.debug
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': settings.db_name,
        'USER': settings.db_user,
        'PASSWORD': settings.db_password.get_secret_value(),
        'HOST': settings.db_host,
        'PORT': settings.db_port,
    }
}
```

---
