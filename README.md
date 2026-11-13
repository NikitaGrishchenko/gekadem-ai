# Gekadem AI

Форк LibreChat с собственным интерфейсом. Ниже — запуск для локальной разработки на Windows через PowerShell.

## Что нужно

- Node.js версии из [.nvmrc](.nvmrc) (сейчас `24.22.3`).
- npm версии из `packageManager` в [package.json](package.json) (сейчас `11.13.0`).
- Docker Desktop с запущенным Docker Engine для локальной MongoDB либо уже работающая MongoDB.

Backend и frontend запускаются локально через npm. [docker-compose.yml](docker-compose.yml) запускает MongoDB, админ-панель, Meilisearch и RAG с PostgreSQL/pgvector.

## Первый запуск

Все команды выполняй из корня проекта:

```powershell
cd C:\projects\gekadem-ai
npm ci
```

### 1. Подготовь Docker-сервисы

Сначала настрой `.env` по следующему разделу, затем запусти сервисы:

```powershell
docker compose up -d --wait
```

Данные сохраняются в каталоге `data-node` внутри проекта, как и в прежнем Compose. Команда ждёт готовности MongoDB; её можно использовать и при следующих запусках. База доступна с компьютера по адресу `127.0.0.1:27017`.

Если ранее запускал отдельный контейнер `gekadem-mongo` по старой инструкции, останови его командой `docker stop gekadem-mongo`, чтобы освободить порт. Его данные остаются в отдельном volume `gekadem-mongo-data` и автоматически в `data-node` не переносятся. Если нужны именно эти данные, запускай прежний контейнер, а остальные сервисы поднимай командой `docker compose up -d --wait admin-panel meilisearch rag_api`.

### 2. Настрой .env

Если `.env` отсутствует, создай его из примера. Существующий файл не перезаписывай:

```powershell
if (!(Test-Path .env)) { Copy-Item .env.example .env }
```

Проверь следующие значения в `.env`:

```dotenv
HOST=0.0.0.0
PORT=3080
MONGO_URI=mongodb://127.0.0.1:27017/LibreChat
DOMAIN_CLIENT=http://localhost:3090
DOMAIN_SERVER=http://localhost:3080
SEARCH=true
MEILI_HOST=http://127.0.0.1:7700
RAG_PORT=8000
RAG_API_URL=http://127.0.0.1:8000
ADMIN_PANEL_URL=http://localhost:3000
ALLOW_REGISTRATION=true
```

Если используешь другую MongoDB, укажи её адрес в `MONGO_URI`. `HOST=0.0.0.0` позволяет админ-панели в Docker обращаться к локальному backend через `host.docker.internal`. Redis не требуется, пока не включены зависящие от него возможности.

Заполни `ADMIN_PANEL_SESSION_SECRET` (не менее 32 символов) и `MEILI_MASTER_KEY` одинаковым ключом для backend и контейнера Meilisearch. Можно сгенерировать отдельные значения командой и перенести результат в `.env`:

```powershell
node -e "const c=require('crypto'); for(const k of ['ADMIN_PANEL_SESSION_SECRET','MEILI_MASTER_KEY']) console.log(k+'='+c.randomBytes(32).toString('hex'))"
```

Админ-панель открывается на **http://localhost:3000** после запуска backend; для входа нужен пользователь с ролью администратора. RAG доступен backend по `http://127.0.0.1:8000`; для обработки документов настрой провайдер embeddings и его ключ в `.env` (например, `EMBEDDINGS_PROVIDER`, `EMBEDDINGS_MODEL`, `RAG_OPENAI_API_KEY`). PostgreSQL доступен RAG внутри Docker, данные сохраняются в volume `pgdata2`. Поиск хранит данные в `meili_data_v1.35.1`.

Если `JWT_SECRET`, `JWT_REFRESH_SECRET`, `CREDS_KEY` и `CREDS_IV` ещё не заполнены, сгенерируй значения:

```powershell
node -e "const c=require('crypto'); for(const k of ['JWT_SECRET','JWT_REFRESH_SECRET','CREDS_KEY','CREDS_IV']) console.log(k+'='+c.randomBytes(k==='CREDS_IV'?16:32).toString('hex'))"
```

Перенеси результат в соответствующие поля `.env`. Уже настроенные значения сохрани: от них зависят сессии и расшифровка сохранённых учётных данных.

Модели и провайдеры настрой в `.env` и [librechat.yaml](librechat.yaml). Для получения ответов от модели нужны рабочие настройки выбранного провайдера. Дополнительные интеграции могут требовать своих сервисов.

Не коммить `.env` и секреты провайдеров.

### 3. Собери проект

```powershell
npm run frontend
```

Команда собирает общие пакеты и frontend. Первая сборка нужна даже для режима разработки: backend читает `client/dist/index.html` при запуске.

### 4. Запусти backend

В первом терминале:

```powershell
npm run backend:dev
```

Дождись запуска сервера на `http://localhost:3080`.

### 5. Запусти frontend

Во втором терминале, тоже из корня проекта:

```powershell
npm run frontend:dev
```

Открой **http://localhost:3090** и зарегистрируй локальный аккаунт.

Это адрес Vite для разработки. Он автоматически обновляет интерфейс после изменений и перенаправляет `/api` и `/oauth` на backend на порту `3080`. Если порт `3090` занят, смотри фактический адрес в выводе Vite.

## Ежедневный запуск

Запусти Docker Desktop, затем сервисы:

```powershell
docker compose up -d --wait
```

В двух отдельных терминалах:

```powershell
npm run backend:dev
```

```powershell
npm run frontend:dev
```

Открывай **http://localhost:3090**, админ-панель — **http://localhost:3000**. Для остановки npm-процессов нажми `Ctrl+C` в каждом терминале. Docker-сервисы останови командой:

```powershell
docker compose down
```

Данные MongoDB, Meilisearch и PostgreSQL при обычном `docker compose down` сохраняются. Не добавляй `-v`, если нужно сохранить volume PostgreSQL.

## Где менять интерфейс

| Путь | Что находится |
| --- | --- |
| `client/src` | Приложение: страницы, компоненты, навигация |
| `client/public/assets` | Статические изображения и другие ресурсы |
| `packages/client` | Общие UI-компоненты и темы |
| `packages/chat` | Общие компоненты, состояние и транспорт чата |
| `client/src/locales/en/translation.json` | Английские ключи локализации для нового текста |
| `librechat.yaml` | Настройки приложения и провайдеров |

Правки в `client/src` подхватывает Vite. При изменениях в общих пакетах запускай соответствующую сборку в режиме наблюдения в дополнительном терминале:

```powershell
npm run build:watch --workspace packages/client
```

```powershell
npm run build:watch --workspace packages/chat
```

После изменения `.env` или `librechat.yaml` перезапусти backend. После обновления зависимостей или исходников общих пакетов выполни `npm ci` и `npm run frontend` заново.

## Если запуск не получается

- Ошибка подключения к MongoDB: проверь, что контейнер запущен, а `MONGO_URI` указывает на доступную базу.
- Не найден `client/dist/index.html` или сборки общих пакетов: выполни `npm run frontend`.
- Ошибки API в браузере: проверь, что backend работает на `3080`, и посмотри его терминал.
- Интерфейс открывается, но модель не отвечает: проверь ключи, адрес и имя модели в конфигурации провайдера.
- Не меняется интерфейс после правки: открой адрес Vite на `3090`; для изменений в `packages/client` или `packages/chat` включи сборку соответствующего пакета.

## Исходный проект

Основано на LibreChat. Условия использования и исходные уведомления об авторских правах сохранены в [LICENSE](LICENSE).
