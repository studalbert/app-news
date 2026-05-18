# app-news

Новостной блог с JWT-авторизацией, комментариями, платными подписками (Stripe) и закреплением постов у подписчиков.

**Стек:** Django 6 + DRF + PostgreSQL + Celery/Redis · Vue 3 + Vite + Pinia + Tailwind · Nginx (prod)

## Структура

```
app-news/
├── backend/          # Django API (apps: accounts, main, comments, subscribe, payment)
├── frontend/         # Vue SPA
├── docker-compose.yml
├── nginx.conf        # reverse proxy (API, static, media, SPA)
└── .env              # переменные для Docker (см. backend/.env.example)
```

## Быстрый старт (production, Docker)

```bash
cp backend/.env.example .env   # заполнить SECRET_KEY, POSTGRES_PASSWORD, Stripe, домен
docker compose up -d --build
```

После старта: сайт на **:80** (HTTPS — через Let's Encrypt в `nginx.conf`).

Сервисы: `db`, `redis`, `static-init` (migrate + collectstatic), `backend`, `celery-worker`, `celery-beat`, `frontend`, `nginx`.

```bash
docker compose exec backend python manage.py createsuperuser
docker compose logs -f backend
```

## Локальная разработка

**Backend** (нужны PostgreSQL и Redis):

```bash
cd backend
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env          # DEBUG=True, DB_HOST=localhost
python manage.py migrate
python manage.py runserver
```

В отдельных терминалах:

```bash
celery -A config worker -l info
celery -A config beat -l info
```

**Frontend:**

```bash
cd frontend
npm install
npm run dev    # http://localhost:5173
```

API по умолчанию: `http://localhost:8000` (`frontend/src/services/api.js`).

## API (префикс `/api/v1/`)

| Модуль | Путь | Назначение |
|--------|------|------------|
| auth | `/auth/` | register, login, logout, profile, JWT refresh |
| posts | `/posts/` | категории, CRUD постов, popular/pinned/featured |
| comments | `/comments/` | комментарии к постам |
| subscribe | `/subscribe/` | планы, подписка, pin/unpin постов |
| payment | `/payment/` | Stripe checkout, webhooks, история платежей |

Админка: `/admin/`. Авторизация API: `Authorization: Bearer <access_token>`.

## Переменные окружения

Шаблон: `backend/.env.example`.

| Группа | Ключи |
|--------|--------|
| Django | `SECRET_KEY`, `DEBUG`, `FRONTEND_URL`, `ALLOWED_HOSTS` (в settings) |
| БД | `POSTGRES_*`, `DB_HOST`, `DB_PORT` |
| Celery | `CELERY_BROKER_URL`, `CELERY_RESULT_BACKEND` |
| Stripe | `STRIPE_PUBLISHABLE_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` |
| Email | `EMAIL_HOST`, `EMAIL_PORT`, `EMAIL_HOST_USER`, `EMAIL_HOST_PASSWORD` |

Для Docker файл `.env` лежит в **корне** репозитория (см. `env_file` в `docker-compose.yml`).

## Фоновые задачи (Celery Beat)

- проверка истёкших подписок (каждый час)
- напоминания об окончании подписки (ежедневно)
- очистка старых платежей и webhook-событий

## Полезные команды

```bash
# Frontend
npm run build && npm run lint

# Backend
python manage.py makemigrations && python manage.py migrate
python manage.py collectstatic --noinput
```
