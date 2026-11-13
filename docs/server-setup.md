# Запуск пользователем на VPS

Все команды VPS выполняет пользователь самостоятельно. При ошибке остановиться. rostof.ru остаётся на другом IP; его не изменяем. Команды рассчитаны на root и установленный Docker.

## 1. GitHub и checkout

Репозиторий: https://github.com/melnikov412/OTUS-Angie.git

AGENTS.md остаётся локально на iMac и не включается в репозиторий. Один раз из корня локального checkout добавить его в локальные исключения Git:

```bash
printf '\n/AGENTS.md\n' >> .git/info/exclude
```

Это исключение не отправляется на GitHub. Если файл уже добавлен в индекс, сначала выполнить `git rm --cached -- AGENTS.md`: файл останется на диске. В текущем состоянии файл ещё не отслеживается, поэтому эта команда не нужна.

На Mac в Angie_on_Ubuntu26:

```bash
git status
git add .gitignore .env.example README.md compose.yaml docker static docs
git diff --cached
git commit -m "Docker Angie static sites and ACME"
git push
```

На VPS:

```bash
mkdir -p ~/projects
cd ~/projects
git clone https://github.com/melnikov412/OTUS-Angie.git otus-angie
cd otus-angie
```

Git должен быть установлен. Если checkout уже есть: перейти в него, проверить git status, при чистом дереве выполнить git pull --ff-only. Дальнейшие команды выполнять из корня checkout.

## 2. Проверить окружение

Пользователь проверяет A/AAAA всех трёх доменов: A должны указывать на новый VPS; если есть AAAA — на его доступный IPv6. Неверная AAAA может мешать выпуску. Проверить CAA, если записи существуют. Внешний firewall должен пропускать TCP 80/443, включая IPv6 при опубликованной AAAA.

```bash
ss -ltnp '( sport = :80 or sport = :443 )'
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

При занятых портах сначала определить владельца. Не останавливать посторонние службы. UFW не включать вслепую; Docker использует собственные правила публикации портов.

## 3. Каталоги и страницы

```bash
install -d -m 0755 /var/www/static
install -d -m 0700 /srv/angie/acme/staging /srv/angie/acme/production
cp -R static/ai7way_ru static/admin_ai7way_ru static/rostof_ai7way_ru /var/www/static/
find /var/www/static/ai7way_ru /var/www/static/admin_ai7way_ru /var/www/static/rostof_ai7way_ru -type d -exec chmod 0755 {} +
find /var/www/static/ai7way_ru /var/www/static/admin_ai7way_ru /var/www/static/rostof_ai7way_ru -type f -exec chmod 0644 {} +
cp .env.example .env
```

Повторное копирование заменяет одноимённые страницы: не использовать после размещения собственного контента без проверки. .env также не перезаписывать при обычном обновлении. ACME-каталоги приватные: master Angie работает от root. При ошибке доступа анализировать UID/GID и логи, не задавать 777.

## 4. Скачать и закрепить образ

```bash
docker compose pull
docker image inspect docker.angie.software/angie:1.12.4-minimal --format '{{range .RepoDigests}}{{println .}}{{end}}'
```

В .env заменить ANGIE_IMAGE на полученное docker.angie.software/angie@sha256:…; записать digest в отчёт. ANGIE_MODE пока http. Конфиги read-only, MIME-файл сохраняется из образа, ACME-состояние записывается на хост.

## 5. HTTP

```bash
docker compose config --quiet
docker compose run --rm --no-deps --entrypoint angie angie -t
docker compose up -d angie
docker compose ps
docker compose logs --tail=100 angie
```

Одноразовый config test не публикует порты. HTTP-режим не обращается к CA. С внешнего компьютера открыть http://ai7way.ru, http://admin.ai7way.ru и http://rostof.ai7way.ru: каждая страница должна содержать своё имя. На VPS проверить виртуальные хосты:

```bash
curl -i --resolve ai7way.ru:80:127.0.0.1 http://ai7way.ru/
curl -i --resolve admin.ai7way.ru:80:127.0.0.1 http://admin.ai7way.ru/
curl -i --resolve rostof.ai7way.ru:80:127.0.0.1 http://rostof.ai7way.ru/
```

Ожидаем 200 и X-Angie-Deployment: docker-http. Локальный ответ не доказывает внешний доступ.

## 6. Staging HTTPS

В .env изменить ANGIE_MODE на staging, затем:

```bash
docker compose config --quiet
docker compose run --rm --no-deps --entrypoint angie angie -t
docker compose up -d --force-recreate angie
docker compose logs --tail=150 angie
```

Angie сам запрашивает общий сертификат трёх имён и обслуживает HTTP challenge на 80. Отдельная папка .well-known не нужна. До получения сертификата TLS может не работать: смотреть логи, не повторять перезапуски непрерывно. Staging-сертификат не доверен браузером.

Для каждого домена проверить curl -kI https://ИМЯ/ (исключение -k допустимо только для staging). Проверить сертификат:

```bash
openssl s_client -connect ai7way.ru:443 -servername ai7way.ru </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

Повторить для поддоменов. Ожидаем все три SAN и заголовок docker-staging. Данные сохраняются в /srv/angie/acme/staging. Не публиковать содержимое ключей.

После успешного выпуска повторить docker compose up -d --force-recreate angie и проверки. Убедиться по логам и сертификату, что состояние сохранилось и новый выпуск из-за потери файлов не понадобился.

## 7. Production HTTPS

Только после успешного staging установить ANGIE_MODE=production в .env. Повторить config/test/up/logs из предыдущего шага. Состояние отдельно хранится в /srv/angie/acme/production.

С внешнего компьютера проверить все три домена обычным curl -I https://ИМЯ/ БЕЗ -k и браузером. Проверить SAN, issuer и сроки через openssl как выше. Ожидаем доверенную цепочку и docker-production. Повторить пересоздание и проверку сохранности. Staging-успех не доказывает production-успех.

HTTP пока оставлен без перенаправления, HSTS не включён. Добавить их отдельным изменением после подтверждения HTTPS.

## Запуск и остановка контейнера

После первичной настройки выполнять на VPS из `~/projects/otus-angie`. Команды используют текущие параметры `.env`:

```bash
# Запустить Angie; при необходимости создать контейнер
docker compose up -d angie

# Проверить состояние и последние логи
docker compose ps
docker compose logs --tail=100 angie

# Остановить Angie, сохранив контейнер и данные ACME
docker compose stop angie

# Снова запустить ранее остановленный контейнер без применения изменений конфигурации Compose
docker compose start angie
```

После изменения `.env`, режима или mounts использовать `docker compose up -d --force-recreate angie` с предварительными проверками из шагов выше. Пока Angie остановлен, сайты недоступны и сертификаты не продлеваются. Не удалять каталоги `/srv/angie/acme`.

## Исправление документации после пробного запуска

На Mac отредактировать README.md и нужные файлы в docs, затем из корня Angie_on_Ubuntu26:

```bash
git status
git diff -- README.md docs
git add README.md docs
git diff --cached
git commit -m "Update Angie setup documentation after trial run"
git push origin HEAD
```

Обычный push обновляет файлы в репозитории и сохраняет историю. Перезаписывать историю через force для исправлений документации не требуется. Для других изменённых файлов добавлять в `git add` их конкретные пути; не включать `.env`, ключи и ACME-состояние.

На VPS пользователь получает обновление самостоятельно:

```bash
cd ~/projects/otus-angie
git status
git pull --ff-only
```

Pull выполнять при чистом рабочем дереве. Изменения одной документации не требуют перезапуска контейнера.

## Обновление и возврат

Код меняется на Mac, commit/push, затем пользователь получает commit на VPS. При изменении содержимого конфигов текущего режима: docker compose exec angie angie -t, затем docker compose exec angie angie -s reload. При смене режима или mounts требуется up -d --force-recreate.

Возврат к учебному HTTP: ANGIE_MODE=http, config test, up -d --force-recreate. HTTPS при этом станет недоступен. Остановка только Angie: docker compose stop angie. Состояние ACME не удалять. Не использовать down -v для переключения. Выключенный Angie не продлевает сертификаты.

## Native — отдельный этап

Сначала подготовить совместимые пакет и конфиги. Остановить Docker Angie, проверить свободные 80/443, затем запускать native. Установка пакета может сама запустить службу. Сохранить имя клиента ai7way, CA, домены, путь /srv/angie/acme/production. Совместимость и права native UID/GID сначала проверить на staging. Контейнерный angie.conf целиком не копировать: пути pid/логов и пользователь требуют проверки. Два процесса не должны одновременно менять ACME-состояние. Native пока не реализован и не проверен.

## Отчёт

Сохранить commit, image digest, версии, config test, Compose ps, ответы всех доменов HTTP/HTTPS, SAN/issuer/сроки и результат пересоздания. Не публиковать .env, ключи, ACME-state. Дополнительные пакетные модули и native остаются незакрытыми требованиями OTUS.
