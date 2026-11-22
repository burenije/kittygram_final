# Kittygram

Kittygram — сервис, в котором можно зарегистрироваться, создать карточку кота с фотографией и достижениями, а также просматривать карточки других пользователей. Django Admin позволяет администрировать пользователей и данные приложения.

Стек — Python, Django, Django REST Framework, Djoser, Gunicorn, React, PostgreSQL 13, Nginx, Docker Compose и GitHub Actions.

## Запуск на localhost

Нужны Docker Engine или Docker Desktop и Docker Compose v2. В корне проекта выполните:

```bash
cp .env.example .env
docker compose up --build -d --wait
```

В PowerShell первую команду можно выполнить как `Copy-Item .env.example .env`.

- Приложение: [http://localhost:9000](http://localhost:9000).
- API: [http://localhost:9000/api/](http://localhost:9000/api/).
- Админка: [http://localhost:9000/admin/](http://localhost:9000/admin/). Создайте администратора командой `docker compose exec backend python manage.py createsuperuser`.

Backend применяет миграции и собирает статику при запуске. Контейнер `frontend` копирует сборку React в том `static` и завершается с кодом 0. Nginx читает `static` и `media`, а PostgreSQL хранит данные в `pg_data`. Команда `docker compose down` останавливает проект с сохранением томов.

## Переменные окружения

| Переменная | Назначение |
| --- | --- |
| `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` | Имя БД, пользователь и пароль. `db` использует их при первом запуске с пустым `pg_data`, Django — для подключения. |
| `DB_HOST`, `DB_PORT` | Адрес и порт PostgreSQL внутри сети Compose. Оставьте `db` и `5432`. Вне Compose при пустом `DB_HOST` Django использует SQLite. |
| `SECRET_KEY` | Секретный ключ Django. Для сервера сгенерируйте его командой `python -c "import secrets; print(secrets.token_urlsafe(50))"`. |
| `ALLOWED_HOSTS` | Разрешённые имена через запятую, без схемы и порта. Для сервера укажите домен Kittygram. |
| `DEBUG`, `TIME_ZONE` | Режим отладки и часовой пояс. На сервере используйте `DEBUG=false`. |
| `TRUST_PROXY_SSL_HEADER` | Установите `true` на сервере, когда внешний Nginx завершает TLS и перезаписывает `X-Forwarded-Proto` для всех запросов к gateway. |
| `KITTYGRAM_PORT` | Порт gateway на loopback-интерфейсе, по умолчанию `9000`. |
| `DOCKERHUB_USERNAME` | Владелец образов для `docker-compose.production.yml`. |
| `KITTYGRAM_IMAGE_TAG` | Версия образов для production, по умолчанию `latest`. CI передаёт SHA проверенного коммита. |

Файл `.env` исключён из Git. Замените примеры пароля БД и ключа Django перед развёртыванием. При смене пароля в существующей базе сначала измените его в PostgreSQL.

## GitHub Actions

Разрешите запуск Actions в форке. В **Settings → Secrets and variables → Actions** добавьте секреты:

| Секрет | Значение |
| --- | --- |
| `DOCKERHUB_USERNAME` | Логин Docker Hub |
| `DOCKERHUB_TOKEN` | Токен Docker Hub с правом записи |
| `TELEGRAM_BOT_TOKEN` | Токен бота от BotFather |
| `TELEGRAM_CHAT_ID` | ID чата, куда бот может писать. Для личного чата сначала откройте бота и нажмите Start. |

Ruff, тесты Django и React запускаются при пуше в любую ветку. Пуш в `main` или `master` после успешных тестов публикует `kittygram_backend`, `kittygram_frontend` и `kittygram_gateway` с тегами `latest` и SHA коммита. Для проверки Практикумом создайте три публичных репозитория Docker Hub с этими именами. Telegram получает сообщение после публикации и, если включён SSH-деплой, после его завершения. Корневой `kittygram_workflow.yml` — копия действующего workflow для проверки Практикумом.

## Развёртывание рядом с Taski

На сервере с Taski нужны Docker Compose v2, свободный локальный порт для Kittygram и отдельный домен, направленный на тот же сервер. Разместите `.env` в `~/kittygram`, укажите `DOCKERHUB_USERNAME`, домен в `ALLOWED_HOSTS` и уникальные секреты. Порт Kittygram должен отличаться от порта Taski.

Добавьте в существующий HTTPS-блок внешнего Nginx для **домена Kittygram**:

```nginx
client_max_body_size 10M;
server_tokens off;

location / {
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_pass http://127.0.0.1:9000;
}
```

Подставьте выбранный `KITTYGRAM_PORT`, если он отличается от 9000. Настройте сертификат для домена, включите `TRUST_PROXY_SSL_HEADER=true` в `.env`, проверьте конфигурацию через `sudo nginx -t` и примените её через `sudo systemctl reload nginx`. Виртуальный хост Taski продолжает направлять запросы на порт Taski.

Для обновления вручную скопируйте `docker-compose.production.yml` в `~/kittygram` и выполните на сервере:

```bash
cd ~/kittygram
docker compose -f docker-compose.production.yml pull
docker compose -f docker-compose.production.yml up -d --wait
```

Для автоматического обновления добавьте переменную репозитория `DEPLOY_KITTYGRAM=true` и секреты `SSH_HOST`, `SSH_USER`, `SSH_KEY` и `SSH_KNOWN_HOSTS`. SSH использует порт 22. Пользователю нужен доступ к Docker и каталогу `~/kittygram`, `SSH_KEY` должен соответствовать ключу в его `authorized_keys`, а `SSH_KNOWN_HOSTS` должен содержать проверенный ключ хоста. Workflow обновит Compose-файл и запустит образы с SHA коммита. Настройку домена, TLS и серверного `.env` выполните один раз перед включением деплоя.

## Проверки

```bash
docker compose exec -T backend python manage.py test
docker run --rm -e CI=true kittygram_frontend npm test -- --watchAll=false --runInBand
docker compose exec -T gateway nginx -t
```

Для полного набора проверок Практикума скопируйте `tests.yml.example` в `tests.yml` и заполните логины, HTTPS-адреса обоих проектов и выданный `EDU_PROVISIONING_ID`. Затем установите зависимости из `backend/requirements.txt`, задайте `SECRET_KEY` для локального процесса и запустите `python -m pytest` из корня. Эти проверки обращаются к публичным сайтам и Docker Hub.
