# Деплой Gekadem AI на свой сервер

## Быстро: собрать образ на компьютере и отправить в GitHub

Запусти Docker Desktop в режиме Linux containers. Открой **Git Bash** в корне проекта и авторизуйся в GitHub Container Registry:

```bash
cd /c/projects/gekadem-ai
docker login ghcr.io -u NikitaGrishchenko
```

В поле Password вставь **GitHub Personal Access Token (classic)** с разрешением `write:packages`, а не пароль от аккаунта. Создать токен можно в GitHub: **Settings → Developer settings → Personal access tokens → Tokens (classic)**.

Собери образ и отправь его одной командой:

```bash
docker buildx build \
  --platform linux/amd64 \
  --file Dockerfile.multi \
  --target api-build \
  --label org.opencontainers.image.source=https://github.com/NikitaGrishchenko/gekadem-ai \
  --tag ghcr.io/nikitagrishchenko/gekadem-ai:latest \
  --push .
```

Команда сама устанавливает зависимости и собирает приложение внутри Docker. Дождись успешного завершения публикации. Образ появится в **GitHub → твой профиль → Packages** под именем `gekadem-ai`.

В `.env` **на сервере** укажи опубликованный образ:

```dotenv
GEKADEM_IMAGE=ghcr.io/nikitagrishchenko/gekadem-ai:latest
```

При следующих сборках используй ту же команду: она обновляет образ `latest`. Значение `GEKADEM_IMAGE` на сервере менять не нужно. Команда рассчитана на сервер `x86_64`; подробности для ARM приведены ниже. Исходники отправляются отдельно через `git push`.

## Полная инструкция

Схема: собираешь свой образ из этого репозитория → публикуешь его в GitHub Container Registry (`ghcr.io`) → сервер скачивает образ и запускает Docker Compose. Исходный код отправляется через `git push`, Docker-образ — через `docker buildx ... --push`. Это две разные операции.

Для сервера подготовлен отдельный [deploy/compose.yml](deploy/compose.yml). Он запускает твой backend с собранным frontend, админ-панель, MongoDB, Meilisearch, RAG с PostgreSQL/pgvector Caddy для HTTPS и WireGuard-шлюз Gluetun. Локальная разработка остаётся в корневом `docker-compose.yml`.

Ниже команды для **Bash**: на компьютере можно использовать Git Bash, на сервере — обычный Linux shell. Пример рассчитан на новый Linux-сервер архитектуры `x86_64` и свободные порты `80/443`. Если они уже заняты reverse proxy, смотри соответствующий раздел ниже.

## 1. Подготовь сервер и домен

Нужны Docker Engine с плагинами Compose и Buildx, SSH-доступ и Python 3 для генерации секретов. Для установки Docker используй [официальную инструкцию для своей ОС](https://docs.docker.com/engine/install/). На компьютере для сборки нужен Docker Desktop в режиме Linux containers.

Проверь на сервере:

```bash
docker version
docker compose version
uname -m
```

В DNS своего домена создай две записи, например:

| Тип | Имя     | Значение               |
| --- | ------- | ---------------------- |
| A   | `chat`  | Публичный IPv4 сервера |
| A   | `admin` | Тот же IPv4 сервера    |

Получатся `chat.example.com` и `admin.example.com`. Замени их дальше на свои адреса. Если есть AAAA-записи, они тоже должны вести на этот сервер по рабочему IPv6.

Разреши входящие TCP `80/443` в firewall сервера и у провайдера. UDP `443` нужен для HTTP/3 и необязателен для обычного HTTPS. SSH-порт также должен оставаться доступным.

Caddy получает и продлевает сертификаты автоматически после настройки DNS и доступности сервера. При первом запуске удобнее использовать прямые DNS-записи без внешнего прокси. [Как работает автоматический HTTPS](https://caddyserver.com/docs/automatic-https).

## 2. Получи доступ к GitHub Container Registry

В GitHub открой **Settings → Developer settings → Personal access tokens → Tokens (classic)** и создай токен для публикации с разрешением `write:packages`. На сервере для скачивания приватного образа достаточно отдельного токена `read:packages` и доступа к пакету. Для GHCR здесь нужен именно токен **classic**. [Документация GitHub](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry).

На компьютере:

```bash
docker login ghcr.io -u NikitaGrishchenko
```

В запрос Password вставь токен GitHub, а не пароль аккаунта. Не записывай токен в команды сборки, Dockerfile или Git.

Путь образа пиши строчными буквами:

```text
ghcr.io/nikitagrishchenko/gekadem-ai:latest
```

## 3. Собери и опубликуй образ

На компьютере, из корня проекта, сначала закоммить нужные изменения и отправь исходники:

```bash
git status
git push origin main
```

Затем собери и отправь образ:

```bash
docker buildx build \
  --platform linux/amd64 \
  --file Dockerfile.multi \
  --target api-build \
  --build-arg BUILD_COMMIT="$(git rev-parse HEAD)" \
  --build-arg BUILD_BRANCH=main \
  --label org.opencontainers.image.source=https://github.com/NikitaGrishchenko/gekadem-ai \
  --tag ghcr.io/nikitagrishchenko/gekadem-ai:latest \
  --push .
```

Сборка устанавливает зависимости и собирает общие пакеты и frontend внутри Docker. Предварительный `npm run frontend` на компьютере для этой команды не нужен. В образ попадут твои логотипы, favicon, переводы и изменения интерфейса.

Сборка использует текущую рабочую директорию, а не автоматически скачанный `main`. Поэтому закоммиченные исходники должны соответствовать собираемой версии. Образ включает и backend, и frontend: это не dev-сервер Vite.

`.env` исключён из контекста сборки, а `librechat.yaml` подключается отдельно при запуске. Цвета и другие настройки из локального `librechat.yaml` нужно перенести на сервер отдельным файлом.

Проверь опубликованный образ:

```bash
docker buildx imagetools inspect ghcr.io/nikitagrishchenko/gekadem-ai:latest
```

Он появится в разделе **Packages** твоего аккаунта GitHub. По умолчанию новый пакет может быть приватным. Если сделаешь его публичным, любой сможет скачать образ вместе с включённым кодом; серверу тогда не понадобится авторизация для скачивания.

Все сборки публикуются под одним тегом `latest`. После публикации серверу нужно скачать обновлённый образ и пересоздать контейнер приложения по командам из раздела 7.

Если `uname -m` на сервере показывает `aarch64`, для своего образа нужен `--platform linux/arm64`. Перед таким деплоем также проверь поддержку этой архитектуры у образов админки и RAG.

## 4. Перенеси файлы на сервер

На сервере создай каталог. Замени `USER` и `SERVER_IP` в командах своим SSH-пользователем и адресом:

```bash
ssh USER@SERVER_IP 'mkdir -p ~/gekadem'
```

На компьютере, из корня проекта:

```bash
scp deploy/compose.yml deploy/Caddyfile deploy/.env.example USER@SERVER_IP:~/gekadem/
scp librechat.yaml USER@SERVER_IP:~/gekadem/librechat.yaml
```

`librechat.yaml` в этом форке игнорируется Git, поэтому копируется с твоего компьютера. Проверь, что там находятся нужные модели, настройки интерфейса и темы. Передача через SSH сохраняет файл вне публичного репозитория.

Дальнейшие команды выполняй **на сервере**:

```bash
ssh USER@SERVER_IP
cd ~/gekadem
cp -n .env.example .env
chmod 600 .env librechat.yaml
```

`cp -n` сохраняет уже существующий `.env`.

Перед первым запуском создай на сервере каталог `~/gekadem/wireguard` и перенеси туда свой конфиг WireGuard под именем `wg0.conf`. На сервере:

```bash
mkdir -p ~/gekadem/wireguard
chmod 700 ~/gekadem/wireguard
```

С компьютера передай конфиг (замени `/path/to/wg0.conf` своим путём):

```bash
scp /path/to/wg0.conf USER@SERVER_IP:~/gekadem/wireguard/wg0.conf
```

На сервере:

```bash
chmod 600 ~/gekadem/wireguard/wg0.conf
```

Параметры и проверки VPN описаны в разделе «Исходящие запросы через WireGuard» ниже.

## 5. Настрой серверный .env

Открой файл:

```bash
nano .env
```

Укажи образ и свои домены:

```dotenv
GEKADEM_IMAGE=ghcr.io/nikitagrishchenko/gekadem-ai:latest
APP_DOMAIN=chat.example.com
ADMIN_DOMAIN=admin.example.com
DOMAIN_CLIENT=https://chat.example.com
DOMAIN_SERVER=https://chat.example.com
ADMIN_PANEL_URL=https://admin.example.com
```

`APP_DOMAIN` и `ADMIN_DOMAIN` — имена без `https://`. В остальных трёх переменных нужен полный HTTPS-адрес.

Внутренние адреса уже настроены: базы доступны по именам `mongodb`, `meilisearch`, `vectordb`. Backend, RAG и админка разделяют сеть `vpn` и обращаются друг к другу через `127.0.0.1` на портах `3080`, `8000` и `3000`. Caddy обращается к `vpn:3080` и `vpn:3000`.

Сгенерируй незаполненные секреты прямо в `.env`. Команда сохраняет уже заполненные значения и не печатает ключи в терминал:

```bash
python3 - <<'PY'
from pathlib import Path
import re
import secrets

env_path = Path('.env')
text = env_path.read_text()
for key in ('JWT_SECRET', 'JWT_REFRESH_SECRET', 'CREDS_KEY', 'CREDS_IV',
            'ADMIN_PANEL_SESSION_SECRET', 'MEILI_MASTER_KEY', 'POSTGRES_PASSWORD'):
    size = 16 if key == 'CREDS_IV' else 32
    text = re.sub(rf'^{key}=[ \t]*$', lambda _: f'{key}={secrets.token_hex(size)}',
                  text, flags=re.MULTILINE)
env_path.write_text(text)
print('Незаполненные секреты сгенерированы.')
PY
```

Эта генерация подходит для новой установки. Если переносишь существующую базу, сначала перенеси её `CREDS_KEY`, `CREDS_IV` и остальные действующие секреты. Иначе сохранённые ключи пользователей не расшифруются. При обновлениях сохраняй `.env`; пароль PostgreSQL должен соответствовать уже созданной базе.

Настрой провайдеров моделей в `.env` и `librechat.yaml`. Для RAG заполни `RAG_OPENAI_API_KEY` и при необходимости `RAG_OPENAI_BASEURL`, либо настрой другого поддерживаемого провайдера embeddings. Сам запуск RAG-контейнера не даёт ему доступа к модели embeddings.

В примере `ALLOW_REGISTRATION=false`: публичная регистрация закрыта, первого пользователя создашь через CLI.

## 6. Запусти приложение

Если образ приватный, авторизуйся **на сервере** тем Linux-пользователем, который запускает Docker:

```bash
docker login ghcr.io -u NikitaGrishchenko
```

Введи токен с `read:packages`. Затем:

```bash
cd ~/gekadem
test -f librechat.yaml
docker compose config --quiet
docker compose pull
docker compose up -d --wait --wait-timeout 300
docker compose ps
curl --fail http://127.0.0.1:3080/health
```

В этом каталоге файл называется `compose.yml`, поэтому `-f` не требуется. На новом сервере все данные создаются заново: локальные пользователи, чаты и файлы автоматически не переносятся.

Открой:

- приложение: `https://chat.example.com`;
- админку: `https://admin.example.com`.

Снаружи публикуются HTTP/HTTPS. Приложение и админка дополнительно доступны только на loopback сервера (`127.0.0.1:3080` и `127.0.0.1:3000`), базы и RAG не публикуют порты на хост.

Для новой пустой базы создай первый аккаунт:

```bash
docker compose exec -w /app api npm run create-user
```

Введи почту, имя, логин и пароль в диалоге. Первый пользователь новой базы получает роль `ADMIN`. Если база уже содержит пользователей, автоматического повышения роли не будет. В админку входи аккаунтом с административными правами.

Проверь отправку и остановку ответа, повторный вход, открытие сохранённого чата, поиск и загрузку документа для RAG.

## 7. Как обновлять приложение

На компьютере повтори сборку из раздела 3 с тем же тегом `latest`. На сервере сначала сделай резервную копию, затем скачай новый образ и пересоздай контейнер. `.env` менять не нужно:

```bash
cd ~/gekadem
docker compose pull api
docker compose up -d --no-deps --force-recreate --wait --wait-timeout 300 api
docker compose ps
curl --fail http://127.0.0.1:3080/health
```

Команды обновляют приложение. Админка, RAG и остальные сервисы продолжают работать с уже скачанными образами.

Если меняешь только цвета или другие настройки `librechat.yaml`, перенеси обновлённый файл на сервер и выполни `docker compose restart api`. Если меняешь `.env`, используй `docker compose up -d`, чтобы контейнеры получили новое окружение.

Данные находятся в Docker volumes и переживают пересоздание контейнеров. Не используй `docker compose down -v`: это удаляет данные проекта.

## Резервная копия перед обновлением

Следующие команды создают согласованную копию баз и файлов с короткой остановкой приложения. На время копирования сайт будет недоступен. Выполняй из `~/gekadem` в Bash:

```bash
(
set -e
backup="$(date -u +%Y%m%dT%H%M%SZ)"
mkdir -p "backups/$backup"
chmod 700 backups "backups/$backup"
docker pull alpine:3
trap 'docker compose up -d --wait --wait-timeout 300' EXIT
docker compose stop caddy admin-panel api rag_api meilisearch

docker compose exec -T mongodb mongodump --db LibreChat --archive --gzip > "backups/$backup/mongodb.archive.gz"
docker compose exec -T vectordb pg_dump -U myuser -d mydatabase -Fc > "backups/$backup/postgres.dump"
cp .env librechat.yaml compose.yml Caddyfile "backups/$backup/"
cp wireguard/wg0.conf "backups/$backup/wg0.conf"

for volume in uploads images app-data meili-data caddy-data caddy-config; do
  docker run --rm \
    -v "gekadem_${volume}:/source:ro" \
    -v "$PWD/backups/$backup:/backup" \
    alpine:3 tar -czf "/backup/${volume}.tgz" -C /source .
done
)
```

Проверь успешное завершение каждого дампа и архива; при ошибках копия неполная. Сохраняй копии вне этого сервера: они содержат чаты, файлы и секреты. Имена volumes в командах соответствуют `name: gekadem` в подготовленном Compose.

Восстановление резервной копии сначала проверь на отдельной установке, не поверх работающих данных.

## Если на сервере уже есть Nginx, Traefik или Caddy

Используй существующий reverse proxy: направь домен приложения на `http://127.0.0.1:3080`, домен админки — на `http://127.0.0.1:3000`. Он должен передавать исходный Host и `X-Forwarded-Proto`, поддерживать WebSocket и не буферизовать SSE-ответы. Настрой HTTPS в нём.

В этом случае запускай конкретные сервисы без встроенного Caddy:

```bash
docker compose up -d --wait --wait-timeout 300 api admin-panel
```

Зависимости поднимутся автоматически. Если ранее запускал встроенный Caddy, останови его через `docker compose stop caddy`. Во всех дальнейших командах запуска также указывай `api admin-panel`, чтобы не занимать порты существующего proxy. Если reverse proxy сам в другом контейнере, его `127.0.0.1` не является хостом: подключи его к нужной Docker-сети и используй адреса сервисов вместо loopback.

## Исходящие запросы через WireGuard, сайт доступен без VPN

Пользователи открывают `https://ai.gekadem.ru` обычным способом. Caddy принимает HTTPS на публичном IP сервера и передаёт запрос приложению. Обращения приложения к моделям, embeddings и другим внешним API выходят через WireGuard:

```text
Браузер → публичный IP сервера → Caddy → приложение
                                           ↓
                                       WireGuard → внешние API
```

DNS домена продолжает указывать на сервер приложения, а не на VPN-сервер. Пользователям не нужен WireGuard-клиент. Доступность самого домена из их сети всё равно необходима.

Для Docker подходит отдельный контейнер Gluetun с твоим WireGuard-конфигом. `api`, `rag_api` и при необходимости `admin-panel` используют его сетевое пространство. Caddy и базы остаются в обычной Docker-сети. SSH, обновления ОС и скачивание образов используют сеть хоста. [Сетевые пространства WireGuard](https://www.wireguard.com/netns/), [подключение контейнеров к Gluetun](https://github.com/qdm12/gluetun-wiki/blob/main/setup/connect-a-container-to-gluetun.md).

В подготовленном `deploy/compose.yml` VPN уже включён для `api`, `rag_api` и `admin-panel`. До первого запуска положи свой действующий конфиг в `wireguard/wg0.conf`. Без него сервисы приложения не запустятся.

### Подключи WireGuard-конфиг

На сервере положи свой конфиг в `wireguard/wg0.conf` рядом с `compose.yml`. Ограничь доступ к каталогу и файлу: права `700` и `600` соответственно. Не добавляй файл с `PrivateKey` и `PresharedKey` в Git или образ приложения.

Пример структуры обычного WireGuard-конфига; адреса и ключи бери из своего файла:

```ini
[Interface]
PrivateKey = YOUR_PRIVATE_KEY
Address = 10.64.0.2/32

[Peer]
PublicKey = YOUR_SERVER_PUBLIC_KEY
Endpoint = YOUR_VPN_ENDPOINT_IP:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Это пример для IPv4. Если нужен IPv6, отдельно настрой адрес, маршруты и firewall для него; иначе исключи прямой IPv6-выход из VPN-пространства. Проверь поддержку параметров своего файла в Gluetun: параметры `wg-quick`, например `PostUp`, нельзя автоматически считать применёнными. Для custom provider документация требует IP-адрес endpoint; AmneziaWG настраивается отдельно. [Настройка custom WireGuard](https://github.com/qdm12/gluetun-wiki/blob/main/setup/providers/custom.md).

### Что уже настроено в Compose

- `vpn` запускает Gluetun с `VPN_SERVICE_PROVIDER=custom` и `VPN_TYPE=wireguard`. Конфиг подключён только для чтения; отсутствующий файл не будет автоматически создан как каталог.
- `api`, `rag_api` и `admin-panel` используют `network_mode: service:vpn` и ждут готовности VPN по healthcheck Gluetun.
- Порты `127.0.0.1:3080` и `127.0.0.1:3000` публикует `vpn`. Собственных публикаций у приложения и админки нет.
- Caddy и базы работают в обычной Docker-сети. Имена и volumes баз сохранены.
- `FIREWALL_INPUT_PORTS=3080,3000` разрешает Caddy обращаться к приложению и админке. `FIREWALL_OUTBOUND_SUBNETS` разрешает обращения к внутренней Docker-подсети. [Firewall Gluetun](https://github.com/qdm12/gluetun-wiki/blob/main/setup/options/firewall.md).

В серверном `.env` задаётся подсеть, по умолчанию:

```dotenv
VPN_DOCKER_SUBNET=172.30.50.0/24
```

Выбери свободную подсеть, которая не пересекается с адресами WireGuard, другими Docker-сетями и сетью сервера. Это значение используется и в настройках Docker-сети, и в исключении firewall. Не указывай `0.0.0.0/0`: это разрешило бы обход VPN для всего интернета.

Внутренние адреса уже изменены:

| Где                                      | Значение                |
| ---------------------------------------- | ----------------------- |
| `api.environment.RAG_API_URL`            | `http://127.0.0.1:8000` |
| `admin-panel.environment.API_SERVER_URL` | `http://127.0.0.1:3080` |
| Caddy: upstream приложения               | `vpn:3080`              |
| Caddy: upstream админки                  | `vpn:3000`              |

Адреса MongoDB, Meilisearch и PostgreSQL остаются `mongodb:27017`, `meilisearch:7700` и `vectordb:5432`. Публичные `DOMAIN_CLIENT`, `DOMAIN_SERVER` и `ADMIN_PANEL_URL` сохраняют HTTPS-домены.

Если reverse proxy установлен на хосте, его адреса `127.0.0.1:3080` и `127.0.0.1:3000` сохраняются: эти порты теперь публикует VPN-контейнер.

### Проверь перед использованием

Сначала проверь `docker compose config --quiet`. При переходе с прежнего Compose без VPN нужно пересоздать сеть в окно обслуживания:

```bash
docker compose config --quiet
docker compose down
docker compose up -d --wait --wait-timeout 300
```

Обычный `down` сохраняет именованные volumes. Не добавляй `-v`. Для этой операции на сервер должны быть перенесены обновлённые `compose.yml`, `Caddyfile` и WireGuard-конфиг.

После запуска проверь внешний IP из контейнеров, например:

```bash
docker compose exec api node -e "fetch('https://api.ipify.org').then(r=>r.text()).then(console.log).catch(()=>process.exit(1))"
docker compose exec rag_api python -c "import urllib.request; print(urllib.request.urlopen('https://api.ipify.org', timeout=10).read().decode())"
```

Оба должны показывать адрес выхода VPN. Дополнительно проверь:

- Сайт и админка открываются с устройства без VPN, ответы приходят потоком.
- Backend видит MongoDB и поиск, RAG видит PostgreSQL; внутренние Docker-имена разрешаются.
- Внешний DNS не уходит напрямую через хост, IPv6 не обходит туннель.
- При контролируемом обрыве связи с WireGuard peer новые внешние запросы блокируются, а не переключаются на IP сервера. Одна успешная проверка IP этого не доказывает.
- После восстановления туннеля приложение снова может отправлять запросы.

После пересоздания самого VPN-контейнера пересоздавай и подключённые к нему `api`, `rag_api`, `admin-panel`, чтобы они использовали актуальное сетевое пространство. Учитывай это при обновлении Gluetun. Обычное обновление только образа `api` выполняется по разделу 7.

Эта схема направляет интернет-трафик выбранных контейнеров через VPN. Отдельные контейнеры инструментов, удалённые исполнители и браузерные запросы напрямую к внешним сайтам требуют собственной настройки маршрута.

## Где смотреть ошибки

```bash
docker compose logs --tail=100 api
docker compose logs --tail=100 admin-panel
docker compose logs --tail=100 rag_api
docker compose logs --tail=100 caddy
docker compose logs --tail=100 vpn
```

- `denied` при скачивании GHCR: проверь тег, доступ к пакету и авторизацию на сервере.
- Образ скачан, но контейнер не запускается: проверь архитектуру сервера и образа.
- `502`: проверь `docker compose ps`, состояние `api` и его `/health`.
- Не выдаётся сертификат: проверь A/AAAA, доступность `80/443` и логи Caddy.
- Админка не пускает: проверь роль пользователя, `ADMIN_PANEL_URL`, `DOMAIN_CLIENT` и секрет сессии.
- Чат работает, а обработка документов нет: проверь RAG, PostgreSQL и провайдера embeddings.

Перед передачей логов убери токены, ключи и содержимое пользовательских запросов.
