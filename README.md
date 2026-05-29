# Kittygram

![Main Kittygram workflow](https://github.com/FlorentsX/kittygram_final/actions/workflows/main.yml/badge.svg)

Социальная сеть для любителей котиков. Пользователи регистрируются, добавляют своих питомцев с фотографиями, указывают их цвет, год рождения и достижения, а также могут просматривать котиков других пользователей.

## Что есть в проекте

- **Регистрация и аутентификация** по токену.
- **CRUD для котиков**: имя, цвет, год рождения, фотография, достижения.
- **Достижения** — отдельная сущность, связанная с котиками через `ManyToMany`.
- **Автоматический расчёт возраста** котика на основе года рождения.
- **Раздача статики и медиа** через Nginx внутри Docker.

## Стек

- Backend: Python 3.12, Django 5.1, Django REST Framework, Djoser
- База данных: PostgreSQL 13
- Frontend: React 18
- Инфраструктура: Docker, Docker Compose, Nginx
- CI/CD: GitHub Actions, Docker Hub

## Архитектура

В проде поднимаются четыре контейнера:

| Сервис   | Образ                          | Назначение                                  |
|----------|--------------------------------|---------------------------------------------|
| `db`      | `postgres:13`                  | База данных                                 |
| `backend` | `florentsx/kittygram_backend`  | Django + Gunicorn на порту 8000             |
| `frontend`| `florentsx/kittygram_frontend` | Сборка React, копирует build в общий volume |
| `gateway` | `florentsx/kittygram_gateway`  | Nginx, проксирует API/admin, отдаёт статику и медиа, публикует порт 9000 |

Запросы идут по цепочке: внешний Nginx сервера (домен + HTTPS) → `127.0.0.1:9000` → `gateway` → либо в `backend:8000` (для `/api/`, `/admin/`), либо отдаёт статику/медиа из общих volume.

## Установка локально

Понадобится **Docker** и **Docker Compose**.

1. Склонировать репозиторий:
   ```bash
   git clone https://github.com/FlorentsX/kittygram_final.git
   cd kittygram_final
   ```

2. Создать файл `.env` в корне проекта по образцу `.env.example`:
   ```
   SECRET_KEY=django-insecure-...
   DEBUG=True
   ALLOWED_HOSTS=localhost 127.0.0.1
   POSTGRES_DB=django
   POSTGRES_USER=django_user
   POSTGRES_PASSWORD=kittygram_password
   DB_HOST=db
   DB_PORT=5432
   ```

3. Поднять контейнеры:
   ```bash
   docker compose up --build
   ```

4. В другом терминале выполнить миграции, собрать статику и перенести её в общий volume:
   ```bash
   docker compose exec backend python manage.py migrate
   docker compose exec backend python manage.py collectstatic --no-input
   docker compose exec backend cp -r /app/collected_static/. /backend_static/static/
   ```

5. Опционально — создать суперпользователя:
   ```bash
   docker compose exec backend python manage.py createsuperuser
   ```

6. Открыть [http://localhost:9000](http://localhost:9000).

## CI/CD

При пуше в любую ветку GitHub Actions:
- прогоняет `flake8` и тесты Django на версиях Python 3.10 / 3.11 / 3.12;
- прогоняет тесты фронтенда.

Дополнительно при пуше в `main`:
- собирает и публикует три образа на Docker Hub (`kittygram_backend`, `kittygram_frontend`, `kittygram_gateway`);
- по SSH копирует `docker-compose.production.yml` на сервер, делает `pull` → `up -d` → `migrate` → `collectstatic`;
- присылает уведомление в Telegram с автором, сообщением коммита и ссылкой на него.

## Примеры запросов к API

Все примеры — для прода: `https://kittygramproj.duckdns.org`. Локально замените базовый URL на `http://localhost:9000`.

### Регистрация

```http
POST /api/users/

{
  "username": "user1",
  "password": "MyStrongPass123",
  "email": "user1@example.com"
}
```

### Получение токена

```http
POST /api/auth/token/login/

{
  "username": "user1",
  "password": "MyPass123"
}
```

Ответ:
```json
{ "auth_token": "abcdef1234567890..." }
```

Дальше в запросах, требующих авторизации, передавайте заголовок:
```
Authorization: Token abcdef1234567890...
```

### Список котиков

```http
GET /api/cats/
Authorization: Token <ваш_токен>
```

### Создание котика

Фотография передаётся в формате base64. Цвет — в HEX, на сервере конвертируется в человекочитаемое имя (если такого имени у цвета нет — вернётся ошибка валидации).

```http
POST /api/cats/
Authorization: Token <ваш_токен>

{
  "name": "Барсик",
  "color": "#000000",
  "birth_year": 2020,
  "image": "data:image/png;base64,iVBORw0KGgoAAAANS..."
}
```

### Создание достижения

```http
POST /api/achievements/
Authorization: Token <ваш_токен>

{
  "achievement_name": "Поймал мышь"
}
```

## Автор

Konstantin Martynov
