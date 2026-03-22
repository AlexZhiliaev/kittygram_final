# Kittygram: backend, инфраструктура и CI/CD

`Kittygram` — сервис для публикации карточек котиков с авторизацией пользователей, загрузкой изображений и хранением достижений каждого питомца.

## Коротко

`Kittygram` в текущем виде — это контейнеризированный Django API с токен-авторизацией, поддержкой изображений в base64, связями достижений у котиков и автоматизированным CI/CD-пайплайном до production-сервера через GitHub Actions.

## Что представляет собой проект

Проект разворачивается в контейнерах и состоит из 4 сервисов:

- `db` — PostgreSQL 13.
- `backend` — Django + DRF API (Gunicorn, порт 8000 внутри сети Docker).
- `frontend` — сборка статики.
- `gateway` — Nginx, единая точка входа (`:9000`), проксирует API и admin в backend.

Схема запросов:

`клиент -> gateway (nginx) -> backend (django/drf) -> db (postgres)`

## Backend: ключевые особенности реализации

### Технологический стек

- Python 3.10
- Django 3.2.3
- Django REST Framework 3.12.4
- Djoser 2.1.0 (регистрация/логин/токены)
- TokenAuthentication (DRF)
- PostgreSQL 13 + `psycopg2-binary`
- Gunicorn 23.0.0
- Pillow (работа с изображениями)
- webcolors (валидация/преобразование цвета)

### API и бизнес-логика

В проекте подключены маршруты:

- `/api/cats/` — CRUD для котиков.
- `/api/achievements/` — CRUD для достижений.
- `/api/users/`, `/api/token/login/`, `/api/token/logout/` и др. — djoser.
- `/admin/` — Django admin.

### Статика и медиа

- `STATIC_ROOT`: `backend_static/static`
- `MEDIA_ROOT`: `media`
- В `nginx` настроена раздача `/media/` из volume `media`.
- В `nginx` настроена раздача `/` (фронтовой статики) из volume `static`.

## Конфигурация окружения

Пример переменных лежит в `.env.example`.

Минимально необходимые переменные:

- `POSTGRES_DB`
- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
- `DB_HOST` (для docker-compose: `db`)
- `DB_PORT` (обычно `5432`)
- `DEBUG` (`True` или `False`)
- `ALLOWED_HOSTS` (через запятую)
- `SECRET_KEY`
- `DOCKER_USERNAME` (обязательно для production)

## Локальный запуск (тестовый, не production)

Для локальной разработки используется файл `docker-compose.yml` (в нём сервисы `backend/frontend/gateway` собираются из локальных Dockerfile).

### 1. Подготовить `.env`

```bash
cp .env.example .env
```

Для Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

### 2. Собрать и поднять контейнеры

```bash
docker compose up -d --build
```

### 3. Применить миграции

```bash
docker compose exec backend python manage.py migrate
```

### 4. Собрать статику

```bash
docker compose exec backend python manage.py collectstatic --no-input
```

### 5. (Опционально) создать администратора

```bash
docker compose exec backend python manage.py createsuperuser
```

После запуска:

- приложение: `http://localhost:9000`
- админка: `http://localhost:9000/admin/`
- API: `http://localhost:9000/api/`

Остановка:

```bash
docker compose down
```

## Production-конфигурация

Для production используется `docker-compose.production.yml`.

Ключевое отличие от локального файла:

- вместо `build` используются образы, собранные в CI (GitHub Actions) и опубликованные в Docker Hub; на сервере при деплое выполняется `docker compose pull` без локальной сборки.

## CI/CD: как устроен GitHub Actions workflow

В проекте настроен GitHub Actions workflow. Он запускается на `push` в ветку `main`.

Конвейер включает следующие этапы:

1. `backend_tests`: установка Python 3.10, установка зависимостей, запуск `flake8 backend/`, запуск `pytest`.
2. `build_backend_and_push_to_docker_hub`: после успешных backend-тестов собирается backend-образ и отправляется в Docker Hub.
3. `frontend_tests`: установка Node.js 18, `npm ci`, `npm run test`.
4. `build_frontend_and_push_to_docker_hub`: после frontend-тестов собирается frontend-образ и отправляется в Docker Hub.
5. `build_gateway_and_push_to_docker_hub`: сборка и отправка образа nginx/gateway в Docker Hub.
6. `deploy`: копирование `docker-compose.production.yml` на сервер, создание `.env` из GitHub Secrets, затем `docker compose pull`, `down`, `up -d`, миграции и `collectstatic`.
7. `send_message`: отправка уведомления в Telegram после успешного деплоя.

### Какие секреты используются в CI/CD

По workflow требуются как минимум:

- `DOCKER_USERNAME`
- `DOCKERHUB_TOKEN`
- `HOST`
- `USER`
- `SSH_KEY`
- `SSH_PASSPHRASE`
- `POSTGRES_DB`
- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
- `SECRET_KEY`
- `ALLOWED_HOSTS`
- `TELEGRAM_TO`
- `TELEGRAM_TOKEN`

## Полезные команды для эксплуатации

Проверить логи контейнеров:

```bash
docker compose logs -f
```

Проверить логи только backend:

```bash
docker compose logs -f backend
```

Перезапустить сервис backend:

```bash
docker compose restart backend
```

## Структура, важная для backend/infra

```text
backend/                       # Django-проект и API
backend/cats/                  # приложение с моделями/сериализаторами/viewset
backend/kittygram_backend/     # settings, urls, wsgi/asgi
nginx/                         # конфиг gateway и Dockerfile
docker-compose.yml             # локальный запуск (сборка из исходников)
docker-compose.production.yml  # production запуск (образы из Docker Hub)
.env.example                   # пример переменных окружения
```
