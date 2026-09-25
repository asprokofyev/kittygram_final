# Kittygram

Проект **Kittygram** — веб-приложение для публикации фотографий котиков с возможностью указывать их достижения. Пользователи могут регистрироваться, добавлять питомцев, загружать фото и присваивать награды. Реализован как SPA на React + REST API на Django.


## Стек технологий

**Backend:**
- Python 3.12
- Django 5.1
- Django REST Framework
- Djoser (аутентификация по токену)
- PostgreSQL 13.10
- Gunicorn

**Frontend:**
- Node.js 18
- React
- http-server (раздача статики)

**Инфраструктура:**
- Docker, Docker Compose
- Nginx (gateway)
- GitHub Actions (CI/CD)
- Docker Hub (хранение образов)


## Переменные окружения

Создайте файл `.env` в корне проекта:

```env
# PostgreSQL
POSTGRES_DB=kittygram
POSTGRES_USER=kittygram_user
POSTGRES_PASSWORD=надёжный_пароль
DB_HOST=db
DB_PORT=5432
SECRET_KEY=сгенерируйте_через_get_random_secret_key
```

## Запуск локально

1. Клонируйте репозиторий и перейдите в его корень:

   ```bash
   git clone <url-репозитория>
   cd kittygram
   ```

2. Создайте `.env` (см. пример выше).

3. Соберите и запустите контейнеры:

   ```bash
   docker compose up -d --build
   ```

4. Примените миграции и соберите статику:

   ```bash
   docker compose exec backend python manage.py migrate
   docker compose exec backend python manage.py collectstatic --no-input
   docker compose exec backend sh -c "cp -r /app/collected_static/. /backend_static/static/"
   ```

5. Создайте суперпользователя для доступа в админку:

   ```bash
   docker compose exec -it backend python manage.py createsuperuser
   ```

6. Приложение доступно по адресам:
   - фронт: `http://127.0.0.1:9000/`
   - админка: `http://127.0.0.1:9000/admin/`
   - API: `http://127.0.0.1:9000/api/`

Остановить:

```bash
docker compose down
```


## Деплой на сервер

### 1. Подготовка образов

Соберите образы и запушьте их на Docker Hub:

```bash
docker login
docker build -t <username>/kittygram_backend:latest ./backend/
docker build -t <username>/kittygram_frontend:latest ./frontend/
docker build -t <username>/kittygram_gateway:latest ./nginx/
docker push <username>/kittygram_backend:latest
docker push <username>/kittygram_frontend:latest
docker push <username>/kittygram_gateway:latest
```

### 2. Подготовка сервера

Установите Docker и Docker Compose:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo usermod -aG docker $USER
```

Создайте каталог для проекта:

```bash
mkdir -p ~/kittygram
```

### 3. Копирование файлов

```bash
scp .env <user>@<server-ip>:~/kittygram/
scp docker-compose.production.yml <user>@<server-ip>:~/kittygram/
```

### 4. Запуск на сервере

```bash
ssh <user>@<server-ip>
cd ~/kittygram
docker compose -f docker-compose.production.yml pull
docker compose -f docker-compose.production.yml up -d
docker compose -f docker-compose.production.yml exec backend python manage.py migrate
docker compose -f docker-compose.production.yml exec backend python manage.py collectstatic --no-input
docker compose -f docker-compose.production.yml exec backend sh -c "cp -r /app/collected_static/. /backend_static/static/"
docker compose -f docker-compose.production.yml exec -it backend python manage.py createsuperuser
```

### 5. Настройка внешнего nginx

Конфиг `/etc/nginx/sites-available/kittygram`:

```nginx
server {
    listen 80;
    server_name <домен> <ip>;

    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass http://127.0.0.1:9000;
    }
}
```

Активация:

```bash
sudo ln -s /etc/nginx/sites-available/kittygram /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

HTTPS через Certbot:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d <домен>
```


## CI/CD

Workflow настроен в `.github/workflows/main.yml` и запускается при пуше в ветку `main`. Что делает pipeline:

1. Проверяет код backend на соответствие PEP8 (flake8).
2. Запускает тесты backend и frontend.
3. Собирает и пушит образы backend, frontend, gateway на Docker Hub.
4. Деплоит проект на сервер по SSH: `pull`, `up -d`, `migrate`, `collectstatic`, перенос статики в volume.
5. Отправляет уведомления в Telegram на каждом этапе.

### Необходимые секреты в GitHub

**Settings → Secrets and variables → Actions** добавьте:

| Секрет | Значение |
|---|---|
| `DOCKER_USERNAME` | логин на Docker Hub |
| `DOCKER_PASSWORD` | пароль или access token Docker Hub |
| `HOST` | IP-адрес сервера |
| `USER` | пользователь на сервере |
| `SSH_KEY` | приватный SSH-ключ для доступа к серверу |
| `SSH_PASSPHRASE` | парольная фраза ключа (если есть) |
| `TELEGRAM_TO` | chat_id для уведомлений |
| `TELEGRAM_TOKEN` | токен бота от BotFather |
| `SECRET_KEY` | Django SECRET_KEY |
| `ALLOWED_HOSTS` | список разрешённых хостов через запятую |


### Как получить TELEGRAM_TO и TELEGRAM_TOKEN

1. Создайте бота через `@BotFather` → `/newbot` → получите токен (это `TELEGRAM_TOKEN`).
2. Напишите боту любое сообщение.
3. Откройте `https://api.telegram.org/bot<токен>/getUpdates` и найдите `chat.id` (это `TELEGRAM_TO`).


## Автор

Проект выполнен в рамках учебного курса Яндекс Практикум.

- GitHub: [asprokofyev](https://github.com/ваш_логин)
- Docker Hub: [asprokofyev1976](https://hub.docker.com/u/ваш_логин)