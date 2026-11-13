# Angie в Docker на Ubuntu 26.04

Репозиторий: https://github.com/melnikov412/OTUS-Angie.git

Три временных домена: ai7way.ru, admin.ai7way.ru, rostof.ai7way.ru. rostof.ru остаётся на другом сервере и не входит в конфигурацию.

Docker на VPS установлен пользователем: Engine 29.9.3, Compose 5.7.1, hello-world успешен. Запуск Angie и выпуск сертификатов на VPS пока не подтверждены.

## Файлы

- compose.yaml — порты 80/443, конфиги и bind mounts.
- docker/etc/angie/{http,staging,production} — три явных режима без генерации конфигов при старте.
- static — исходники трёх страниц; на VPS копируются в /var/www/static.
- .env.example — параметры, копировать в .env (вне Git).
- docs/server-setup.md — последовательные команды для пользователя.

Образ 1.12.4-minimal закреплён digest в .env.example. На VPS проверить загрузку и digest по инструкции. Встроенного ACME достаточно; дополнительные модули OTUS и native-установка остаются отдельным этапом.

MIME-файл остаётся из образа: весь /etc/angie не перекрывается. Конфиги и статика подключены read-only, ACME-каталог — rw. Логи отправляются в stdout/stderr с ротацией Docker. HTTP пока не перенаправляется на HTTPS; HSTS включим после успешной проверки доверенного сертификата.

## Начало работы

Следовать [серверной инструкции](docs/server-setup.md). Все команды VPS выполняет пользователь; Codex не подключается к серверу. Не выполнять команды сервера на Mac.

## Запуск и остановка

После первичной настройки по серверной инструкции, из корня checkout на VPS:

```bash
docker compose up -d angie
docker compose ps
docker compose logs --tail=100 angie
docker compose stop angie
```

Повторный запуск остановленного контейнера: `docker compose start angie`. После изменения `.env` или режима следовать проверкам и пересозданию из [серверной инструкции](docs/server-setup.md). Пока контейнер остановлен, сайты недоступны и сертификаты не продлеваются.

AGENTS.md хранится только локально на iMac; порядок исключения из Git и обновления документации приведён в серверной инструкции.

## Источники

- https://angie.software/angie/docs/installation/docker/
- https://angie.software/angie/docs/configuration/modules/http/http_acme/
- https://docs.docker.com/engine/install/ubuntu/

## Проверки

Состояние локальных проверок записано в docs/validation.md. Локальный синтаксический тест не подтверждает DNS, firewall, сертификаты и доступность VPS.
