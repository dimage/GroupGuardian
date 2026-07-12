# 🤖 GroupGuardian — концепция Telegram-бота-модератора

## Часть 2: Модели данных и настройка окружения

---

## 🔧 Переменные окружения (`.env`)

### Обязательные переменные

| Переменная | Описание | Пример |
|------------|----------|--------|
| `BOT_TOKEN` | Токен бота от @BotFather | `123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11` |
| `DB_HOST` | Хост PostgreSQL | `localhost` или `postgres` (в Docker) |
| `DB_PORT` | Порт PostgreSQL | `5432` |
| `DB_USER` | Пользователь PostgreSQL | `postgres` |
| `DB_PASSWORD` | Пароль PostgreSQL | `strong_password` |
| `DB_NAME` | Имя базы данных | `telegram_bot` |
| `REDIS_HOST` | Хост Redis | `localhost` или `redis` (в Docker) |
| `REDIS_PORT` | Порт Redis | `6379` |
| `REDIS_DB` | Номер базы Redis | `0` |

### Опциональные переменные

| Переменная | Описание | Значение по умолчанию |
|------------|----------|----------------------|
| `RATE_LIMIT` | Лимит запросов в минуту | `30` |
| `DEBUG` | Режим отладки | `False` |
| `DEFAULT_LANG` | Язык по умолчанию | `ru` |
| `WARN_EXPIRY_DAYS` | Срок действия предупреждений (дней) | `30` |
| `DEFAULT_WARN_LIMIT` | Лимит предупреждений до бана | `3` |
| `DEFAULT_MUTE_DURATION` | Время мута по умолчанию | `10m` |
| `ANTI_RAID_THRESHOLD` | Порог для защиты от рейдов | `10` |
| `ANTI_RAID_WINDOW` | Временное окно (минут) | `5` |
| `CAPTCHA_TIMEOUT` | Таймаут на капчу (секунд) | `60` |
| `CAPTCHA_BAN_DURATION` | Длительность первого бана за ошибку (в минутах) | `10` |
| `CAPTCHA_ENABLED` | Включена ли капча по умолчанию для новых чатов | `True` |
---

## 📊 Структура базы данных (SQLAlchemy)

### Схема таблиц

```sql
-- Пользователи
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    telegram_id BIGINT UNIQUE NOT NULL,
    username VARCHAR(64),
    first_name VARCHAR(128),
    last_name VARCHAR(128),
    is_bot BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Чаты
CREATE TABLE chats (
    id SERIAL PRIMARY KEY,
    chat_id BIGINT UNIQUE NOT NULL,
    title VARCHAR(256),
    type VARCHAR(32) DEFAULT 'group',
    language VARCHAR(2) DEFAULT 'ru',
    warn_limit INTEGER DEFAULT 3,
    mute_duration VARCHAR(10) DEFAULT '10m',
    warn_on_media BOOLEAN DEFAULT TRUE,
    moderate_enabled BOOLEAN DEFAULT FALSE,
    antiraid_enabled BOOLEAN DEFAULT TRUE,
    autoclean_days INTEGER DEFAULT NULL,
    captcha_enabled BOOLEAN DEFAULT TRUE,
    captcha_timeout_seconds INTEGER DEFAULT 60,
    captcha_ban_duration_minutes INTEGER DEFAULT 10,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Предупреждения
CREATE TABLE warnings (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    chat_id INTEGER REFERENCES chats(id) ON DELETE CASCADE,
    admin_id INTEGER REFERENCES users(id),
    reason TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE
);

-- Ограничения пользователей (/nomedia, /shadowban)
CREATE TABLE user_restrictions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    chat_id INTEGER REFERENCES chats(id) ON DELETE CASCADE,
    type VARCHAR(32) NOT NULL, -- 'nomedia', 'shadowban'
    admin_id INTEGER REFERENCES users(id),
    expires_at TIMESTAMP, -- NULL = бессрочно
    created_at TIMESTAMP DEFAULT NOW()
);

-- Задания для капчи (слово → эмодзи)
CREATE TABLE captcha_tasks (
    id SERIAL PRIMARY KEY,
    word VARCHAR(64) UNIQUE NOT NULL,
    emoji VARCHAR(8) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Сессии капчи для отслеживания проверок
CREATE TABLE captcha_sessions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    chat_id INTEGER REFERENCES chats(id) ON DELETE CASCADE,
    task_id INTEGER REFERENCES captcha_tasks(id),
    message_id INTEGER NOT NULL,
    attempts INTEGER DEFAULT 0,
    is_passed BOOLEAN DEFAULT FALSE,
    is_expired BOOLEAN DEFAULT FALSE,
    started_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,
    UNIQUE(user_id, chat_id)  -- активная сессия только одна на пользователя в чате
);

-- Нарушения капчи для прогрессивного бана
CREATE TABLE user_violations (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    chat_id INTEGER REFERENCES chats(id) ON DELETE CASCADE,
    violation_type VARCHAR(32) NOT NULL,  -- 'captcha_fail'
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,  -- NULL = бессрочный бан
    is_active BOOLEAN DEFAULT TRUE
);

-- Запрещённые слова
CREATE TABLE banned_words (
    id SERIAL PRIMARY KEY,
    chat_id INTEGER REFERENCES chats(id) ON DELETE CASCADE,
    word VARCHAR(128) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(chat_id, word)
);

-- Роли администраторов
CREATE TABLE admin_roles (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    chat_id INTEGER REFERENCES chats(id) ON DELETE CASCADE,
    role VARCHAR(32) DEFAULT 'moderator', -- 'owner', 'admin', 'moderator'
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id, chat_id)
);

-- Кастомные команды
CREATE TABLE custom_commands (
    id SERIAL PRIMARY KEY,
    chat_id INTEGER REFERENCES chats(id) ON DELETE CASCADE,
    key VARCHAR(64) NOT NULL,
    response TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(chat_id, key)
);

-- Логи действий администраторов
CREATE TABLE admin_logs (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    chat_id INTEGER REFERENCES chats(id),
    command VARCHAR(64),
    params TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Жалобы пользователей
CREATE TABLE reports (
    id SERIAL PRIMARY KEY,
    reporter_id INTEGER REFERENCES users(id),
    reported_id INTEGER REFERENCES users(id),
    chat_id INTEGER REFERENCES chats(id),
    reason TEXT,
    status VARCHAR(16) DEFAULT 'pending', -- 'pending', 'resolved', 'rejected'
    created_at TIMESTAMP DEFAULT NOW(),
    resolved_at TIMESTAMP
);

-- Голосования
CREATE TABLE polls (
    id SERIAL PRIMARY KEY,
    chat_id INTEGER REFERENCES chats(id) ON DELETE CASCADE,
    admin_id INTEGER REFERENCES users(id),
    question TEXT NOT NULL,
    options JSONB NOT NULL,
    message_id INTEGER,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),
    closed_at TIMESTAMP
);

-- Приветствия и прощания
CREATE TABLE welcome_messages (
    chat_id INTEGER PRIMARY KEY REFERENCES chats(id) ON DELETE CASCADE,
    message TEXT,
    is_enabled BOOLEAN DEFAULT TRUE
);

CREATE TABLE goodbye_messages (
    chat_id INTEGER PRIMARY KEY REFERENCES chats(id) ON DELETE CASCADE,
    message TEXT,
    is_enabled BOOLEAN DEFAULT TRUE
);

-- Защита от рейдов
CREATE TABLE raid_protection (
    chat_id INTEGER PRIMARY KEY REFERENCES chats(id) ON DELETE CASCADE,
    threshold INTEGER DEFAULT 10,
    window_minutes INTEGER DEFAULT 5,
    is_active BOOLEAN DEFAULT TRUE,
    mute_duration VARCHAR(10) DEFAULT '1h'
);
```

## 🗄 Интеграция с внешними сервисами

| Сервис | Назначение | Подключение |
|--------|------------|-------------|
| **Redis** | Кэширование, TTL для ограничений и сессий капчи | Через `REDIS_HOST`, `REDIS_PORT`, `REDIS_DB` |
| **Grafana** | Визуализация метрик | Подключение к Prometheus |
| **Prometheus** | Сбор метрик | Экспорт метрик через `prometheus_client` |
| **Amplitude / PostHog** | Аналитика поведения | Через API-ключи в `.env` |
| **Sentry** | Отслеживание ошибок | Через DSN в `.env` |

---

## 🔗 Ссылки

- Документация SQLAlchemy: [docs.sqlalchemy.org](https://docs.sqlalchemy.org/)
- Документация Alembic: [alembic.sqlalchemy.org](https://alembic.sqlalchemy.org/)
- Документация Pydantic: [docs.pydantic.dev](https://docs.pydantic.dev/)
