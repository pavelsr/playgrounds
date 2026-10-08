# AGENTS.md

Этот репозиторий — набор папок с Docker Compose-конфигами тестовых серверов СУБД (PostgreSQL, ClickHouse и другие) для экспериментов и учёбы.

Каждая папка — один стек. Новый датасет той же СУБД — не новая папка и не новый оверлей, а запись в конфиге уже существующего стека, см. «Не изобретай велосипед». Оверлей — для того, что ссылкой на датасет не выражается.

Правила контейнеров ниже совпадают с репозиторием dockerfiles (`/home/a/dockerfiles/AGENTS.md`). Отличия этого репозитория — в разделах «Запуск», «Данные» и «Init-контейнеры»: здесь СУБД является самим сервисом, а не внешней зависимостью.

## Запуск в одну команду

Любая конфигурация поднимается ровно одной командой `docker compose up`. Отдельные скачивания, `psql`, `clickhouse-client`, `docker compose run` и второй `up` для данных запрещены.

Базовый стек:

```bash
docker compose up -d
```

Стек с оверлеем — базовый файл первым, оверлеи дальше, в том порядке, в каком они должны сливаться:

```bash
docker compose -f docker-compose.yaml -f docker-compose.pagila.yaml up -d
```

Несколько оверлеев — несколько `-f` в той же одной команде. Поздний файл перекрывает ранний.

Не рассчитывай на неявный `docker-compose.override.yaml`. Compose подхватывает его только когда `-f` не передан. В команде и в doco-cd перечисляй файлы явно. Coolify оверлеи сам не сливает: ему нужен один полный файл, см. раздел ниже.

Относительные пути во всех файлах (volumes, `env_file`, `build`) считаются от каталога базового `docker-compose.yaml`. Оверлей одного стека лежит в его папке. Оверлей, который подключают несколько стеков, лежит в `common/`, см. раздел «Общие оверлеи».

## Совместимость с doco-cd и Coolify

Один и тот же compose-файл обязан подниматься локально, в [doco-cd](https://doco.cd/) и в [Coolify](https://github.com/coollabsio/coolify). Конфиг, которому нужна вторая команда, ручной файл на хосте или ключ вне спецификации Compose, сломан для обоих.

### doco-cd

doco-cd клонирует репозиторий и делает `up` по списку файлов.

- В папке ровно один файл из имён, которые doco-cd ищет по умолчанию: `docker-compose.yaml`. Не добавляй рядом `compose.yaml`, `compose.yml`, `docker-compose.yml` — при автодискавери doco-cd сольёт все найденные.
- Оверлей одного стека называй `docker-compose.<имя>.yaml` (`docker-compose.pagila.yaml`). Общий оверлей — короткое имя в `common/` (`prod-nets.yml`). Оба варианта не входят в список имён по умолчанию и не станут отдельным стеком при `auto_discovery`. В `common/` не клади `docker-compose.yaml`, `compose.yaml` и их `.yml`-варианты.
- В `.doco-cd.yml` тот же порядок, что и в команде:

```yaml
name: postgres-pagila
working_dir: 01-postgres
compose_files:
  - docker-compose.yaml
  - docker-compose.pagila.yaml
```

Несуществующие файлы doco-cd пропускает. Базовый файл существовать обязан.

- Стек — Compose-проект, не Swarm. В Swarm `depends_on` игнорируется, init-контейнер не отработает. Блок `swarm` для этих папок не задавай.
- Сервисы с `profiles` при деплое doco-cd не стартуют, пока профиль не указан в `profiles` деплоя. Датасет и init в profile не выноси.
- Переменные интерполяции имеют дефолт в compose-файле (`${APP_PORT:-5432}`). Отсутствующий `.env` doco-cd игнорирует. `up` без предварительного `cp .env.example .env` обязан работать.
- Init, который завершается с кодом 0, doco-cd считает успешным только если долгоживущий сервис ждёт его через `condition: service_completed_successfully`. Иначе вышедший контейнер выглядит как упавший деплой.
- Тяжёлый датасет может не уложиться в таймаут деплоя doco-cd (по умолчанию 180 секунд). Поднятие стека всё равно одна команда `up`; для такого датасета в деплое doco-cd подними `timeout`. В README напиши оценку времени загрузки.

### Coolify

Coolify для Git-репозитория — build pack Docker Compose. Base Directory — `/<папка>`, Docker Compose Location — `/docker-compose.yaml`. Coolify читает только этот файл: оверлеи из `-f` и директиву `include` при разборе он не подхватывает.

Режим деплоя — **Raw (deploy file as-is)** (Advanced → Docker compose → Compose deployment). В обычном режиме Coolify переписывает `container_name`, тома и сети и подключает свой прокси. Raw запускает файл как есть, тем же составом, что локальный `up` и doco-cd. Прокси-лейблы Coolify для СУБД не добавляй: к базе подключаются по опубликованному хост-порту.

- Базовый `docker-compose.yaml` самодостаточен. Фрагмент оверлея (только `dbinit` без сервиса СУБД) в Coolify не деплоится. Вариант, который должен жить в Coolify отдельно от базы, — либо полный compose-файл в Docker Compose Location, либо своя папка с полным `docker-compose.yaml`. Оверлей-фрагмент оставляй для локальной команды и `compose_files` doco-cd и напиши это в README.
- В файл не добавляй ключи Coolify вне спецификации Compose: `exclude_from_hc`, `content`, `is_directory`. `docker compose` и doco-cd такой файл не примут. Одноразовый init и так выпадает из проверки живости: `restart: on-failure` и `depends_on.<init>.condition: service_completed_successfully`.
- У долгоживущего сервиса есть `healthcheck` по разделу «Healthcheck». По нему Coolify видит, что сервис поднялся.
- Интерполяция — `${VAR:-значение}`. Форма `${VAR:?}` заставляет Coolify блокировать деплой, пока переменную не заполнят в UI. Для учебных дефолтов она не нужна.
- Домен и `SERVICE_URL_*` / `SERVICE_FQDN_*` для СУБД не задавай. HTTP-порт (например ClickHouse `8123`) публикуй на хост, как и нативный. Через прокси Coolify базу не выводи.
- Сеть провайдера (`pg-net`, `clickhouse-net`) в файле остаётся. Внутри стека сервисы ходят по имени сервиса. Снаружи, в том числе из другого compose на том же хосте, — по `container_name` и этой сети. В Coolify в Raw имена не переписываются.
- Bind mount файла из репозитория: в Coolify включи Preserve repository during deployment, иначе файл в каталоге деплоя пропадёт и Docker создаст на его месте каталог.

## Общие оверлеи

Оверлей, который нужен нескольким стекам, клади в `common/` в корне репозитория. Оверлей одного стека остаётся в его папке как `docker-compose.<имя>.yaml`.

Имя файла в `common/` короткое, без префикса `docker-compose.`. Оверлей подключения к прокси-сети — `prod-nets.yml`. Другие файлы называй так же: зачем оверлей, два-три слова, расширение `.yml`.

Запуск по-прежнему одна команда. База первая, общий оверлей следующим `-f`:

```bash
docker compose -f docker-compose.yaml -f ../common/prod-nets.yml up -d
```

В doco-cd тот же путь относительно `working_dir` стека: `../common/prod-nets.yml`. Coolify этот файл сам не подхватит: ему по-прежнему нужен один полный compose.

Файлы `common/` лежат в git. Проект ведёт деплой как GitOps: doco-cd и Coolify берут конфигурацию из репозитория, а не с машины, где её один раз поправили руками.

Папку `common/` с нуля создавай сразу с `README.md`. В нём — что за оверлеи лежат в папке и зачем каждый нужен, и что они в репозитории, потому что проект придерживается GitOps. Не больше 30 строк. Перед сохранением посчитай строки и уложись в лимит. Появился новый общий оверлей — допиши его в этот README в тот же лимит, а не заводи второй файл.

В `common/` нет `.env` и нет `.env.example`. Значения, которые отличаются от стека к стеку (`VIRTUAL_HOST` и подобные), задавай в `.env` / `.env.example` папки стека. Оверлей только подставляет их:

```yaml
services:
  db:
    environment:
      VIRTUAL_HOST: ${VIRTUAL_HOST}
```

Compose берёт `${VIRTUAL_HOST}` из `.env` каталога первого `-f`, то есть из папки стека. Не указывай в общем оверлее `env_file` на `common/.env` и не ставь `--project-directory` в `common/`. Дефолт в самом оверлее (`${VIRTUAL_HOST:-...}`) допустим только как запасной, когда в `.env` стека переменной нет. Разные стеки не должны получать один и тот же хост из файла в `common/`.

## Данные

Файлы баз и датасетов в репозиторий не клади: ни дампы, ни CSV, ни Parquet, ни бэкапы, ни каталоги данных — даже если файл маленький.

Храни ссылку на скачивание с внешнего ресурса. Если есть зеркало — храни и его. Куда класть ссылки и как передать несколько датасетов — в разделе «Не изобретай велосипед». Нет зеркала — в README одна фраза, что зеркала нет, и зеркало в конфиг не выдумывай.

Скачанное лежит в named volume, не в каталоге проекта.

## Init-контейнеры

Тестовые данные готовит init-контейнер. Он отрабатывает до основного сервиса и завершается с кодом 0. Основной сервис стартует только после этого.

Init не подключается к уже запущенной СУБД. Он кладёт файлы в volume, основной сервис при первом старте сам их применяет (у официальных образов PostgreSQL и ClickHouse это каталог `/docker-entrypoint-initdb.d`).

```yaml
services:
  dbinit:
    image: busybox:1.37
    container_name: ${DBINIT_CONTAINER_NAME:-postgres-dbinit}
    environment:
      DATASET_URLS: ${DATASET_URLS:-https://example.com/pagila.sql}
    volumes:
      - init-sql:/init
    restart: on-failure
    command:
      - sh
      - -c
      - |
        set -eu
        n=0
        printf '%s\n' "$$DATASET_URLS" | tr ',' '\n' | while IFS= read -r url; do
          [ -n "$$url" ] || continue
          n=$$((n + 1))
          wget -O "/init/$$(printf '%02d' "$$n")-data.sql" "$$url"
        done

  db:
    image: postgres:17-alpine
    container_name: ${DB_CONTAINER_NAME:-postgres}
    restart: unless-stopped
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - init-sql:/docker-entrypoint-initdb.d:ro
    depends_on:
      dbinit:
        condition: service_completed_successfully
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  init-sql:
  postgres-data:
```

Правила init:

- Имя сервиса — одно слово: `dbinit`, не `postgres-init`.
- У init есть `container_name` через переменную `<SERVICE>_CONTAINER_NAME`.
- `restart: on-failure`. После кода 0 контейнер стоит. При ошибке скачивания или распаковки Docker его повторяет. `restart: unless-stopped` на init запрещён. `restart: "no"` здесь не ставь: init — часть `up`, а не `docker compose run`.
- Init не в `profiles`. Профили — только по разделу «profiles».
- Долгоживущий сервис, которому нужны данные, содержит `depends_on.<init>.condition: service_completed_successfully`. Короткий список `depends_on` для этого ожидания не подходит: он не ждёт код 0.
- Скрипт init — в `command` / `entrypoint` сервиса или в `entrypoint.sh`, если в одну оболочку он уже нечитаем. Отдельный `.sh` на каждый датасет не заводи.
- Повторный `up` при уже инициализированном data volume не должен падать. Официальный entrypoint PostgreSQL выполняет `/docker-entrypoint-initdb.d` только на пустом каталоге данных. Не делай init, который считает ошибкой уже лежащие файлы.
- `$$` в shell-скрипте внутри compose — литерал `$` для оболочки. Одинарный `$` Compose съест как свою переменную.

Разовые задачи, которые не готовят данные основного сервиса (отладка, бенчмарк), запускай через `docker compose run --rm <сервис>`, с `restart: "no"`. В `profiles` их не прячь. В README указывай эту команду `run`.

## Структура

Каждая папка в корне, кроме `common/`, начинается с двузначного номера и дефиса: от `01-` до `99-`. Номер уникален. Следующая папка берёт ближайший свободный номер. `00-` не используй. `common/` остаётся без номера: это не стек, а общий каталог оверлеев.

Имя после номера — сервис: `01-postgres`, `02-clickhouse`. Префикс в `container_name`, в имя сети и в имя сервиса не входит.

Суффиксы те же, что в репозитории dockerfiles:

- `-min` — в compose используется SQLite;
- `-pgext` — своего сервера БД в файле нет, подключение к внешнему PostgreSQL;
- `-<hostname>` — конфиг только этого хоста (локальные домены, IP). Общие шаблоны (`caddy`, `dnsmasq`) без суффикса.

Новый датасет для уже существующего сервера добавляй в его конфиг, см. «Не изобретай велосипед». Второй стек той же СУБД заводи только если нужна другая мажорная версия или два экземпляра одновременно. У параллельного экземпляра свои `container_name`, хост-порты и `DB_NETWORK_NAME`, чтобы не занять `pg-net` и порты первого.

В каждой папке сервиса есть `README.md`. В `common/` свой короткий README, см. «Общие оверлеи».

В папке сервиса допустимы только:

- `docker-compose.yaml`
- `docker-compose.<оверлей>.yaml`
- `Dockerfile`
- `.env.example`
- `README.md` (не больше 100 строк)
- `entrypoint.sh` — только если у образа свой entrypoint
- текстовые конфиги — только если переменных окружения не хватает
- `datasets.conf` — только когда датасетов больше 10, см. «Не изобретай велосипед»

## container_name

У каждого сервиса задан `container_name`.

Короткое имя по умолчанию — имя папки после префикса `NN-` и до следующего `-`. Папка `01-postgres` → `postgres`. Папка `02-clickhouse-min` → `clickhouse`. Номер папки в `container_name` не попадает.

- Один сервис — переменная `CONTAINER_NAME`.
- Несколько сервисов — `<SERVICE_NAME>_CONTAINER_NAME` у каждого. Имя сервиса в переменной — в верхнем регистре: сервис `db` → `DB_CONTAINER_NAME`.

Имя сервиса (`services.<имя>`) — одно слово: `db`, `dbinit`, `web`.

```yaml
services:
  db:
    container_name: ${DB_CONTAINER_NAME:-postgres}
  dbinit:
    container_name: ${DBINIT_CONTAINER_NAME:-postgres-dbinit}
```

```dotenv
DB_CONTAINER_NAME=postgres
DBINIT_CONTAINER_NAME=postgres-dbinit
```

## docker-compose.yaml

Долгоживущим сервисам, которые поднимает `up`, ставь `restart: unless-stopped`.

### Healthcheck

Если у базового образа нет своего `HEALTHCHECK`, задай `healthcheck` в compose каждому долгоживущему сервису. Образ не проверяй на глаз: после `docker pull` посмотри `docker inspect --format '{{json .Config.Healthcheck}}' <образ>`. `null` — проверки в образе нет, её пишет compose.

Проверка отвечает на вопрос «сервис принимает подключения», а не «процесс запущен». Для PostgreSQL это `pg_isready`, для ClickHouse — HTTP `ping`. Команда есть в том образе, который указан в `image`. Интервал, timeout и retries задавай явно.

Одноразовый init не проверяй: он должен завершиться с кодом 0, а не стать healthy. Его ждёт `condition: service_completed_successfully`.

Свой `HEALTHCHECK` образа не дублируй в compose, пока поведение не нужно заменить.

### profiles

`docker compose --profile` и ключ `profiles` у сервиса — крайняя мера. Сначала убери лишний сервис из файла или запусти его отдельно через `docker compose run --rm`.

Профиль допустим только если сервис обязан лежать в том же compose-файле и при обычном `up` его старт ломает стек, а `run` этого не выражает. Датасет, init и всё, без чего стек не готов к работе, в профиль не выноси: doco-cd и Coolify при деплое профиль не включают.

Если профиль всё же есть, в README напиши, почему без него нельзя, и одну команду `docker compose --profile <имя> up -d`.

`network_mode: host` не используй. Доступ к сервису на хосте — через `host.docker.internal`:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

`network_mode: host` — только если пользователь явно попросил или без него сервис не работает.

Данные — в named volumes. Bind mount — только для конфигов из папки проекта, с флагом `:ro`.

```yaml
services:
  db:
    image: postgres:17-alpine
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

Не монтируй каталог данных с хоста (`./data:/var/lib/postgresql/data`).

### Порты

Внутри контейнера веб-сервер слушает `80`, если пользователь не задал другой порт. Внутренний порт — `VIRTUAL_PORT`, дефолт `80`.

Имя переменной хост-порта:

- один веб-сервис при обычном `up` — `APP_PORT`;
- два и больше веб-сервисов без profile — `<PROJECT_SLUG>_PORT`.

СУБД — не веб-сервер. Хост-порт выноси в переменную с дефолтом: `POSTGRES_PORT`, `CLICKHOUSE_HTTP_PORT`, `CLICKHOUSE_NATIVE_PORT`. Дефолт уникален среди всех `.env.example` и `docker-compose.yaml` репозитория. Перед добавлением порта сверься с уже занятыми.

```yaml
ports:
  - "${POSTGRES_PORT:-5432}:5432"
```

Существующие конфиги под это правило не переписывай, пока пользователь не попросил. Для новых — обязательно.

### Сети

Критерий — подключаются ли другие compose-проекты к сети стека (`external: true`).

**Standalone** — снаружи только published ports. Блок `networks` не пиши, Compose подключит `default`.

**Провайдер** — другие стеки ходят в этот сервис по Docker-сети. Одна явная сеть, сервис подключён только к ней, не к `default` параллельно.

Имя сети на хосте:

- обычный сервис из одного слова — `<container_name>-net` (`clickhouse` → `clickhouse-net`);
- PostgreSQL — `pg-net`;
- многословная конфигурация — имя папки без префикса `NN-` и без суффикса `-net`.

Имя сети — в `.env.example`. Для БД переменная `DB_NETWORK_NAME`.

```yaml
services:
  db:
    networks:
      - db-net

networks:
  db-net:
    name: ${DB_NETWORK_NAME:-pg-net}
```

Несколько локальных сервисов в одном файле, которым нужно видеть друг друга, — у каждого явно `networks: - default` (и другие сети по необходимости).

Имена вида `{папка}_default` не используй.

Тестовый PostgreSQL из этой папки — провайдер для стеков из репозитория dockerfiles: те подключаются к `pg-net` как `external: true` и ходят по `container_name`.

## Образы

Базовый образ — официальный образ проекта. Бери его с реестра, который публикует сам проект: [Docker Hub](https://hub.docker.com/), [ghcr.io](https://ghcr.io/), [registry.gitlab.com](https://registry.gitlab.com/), [quay.io](https://quay.io/) или другой реестр того же класса. В `image` и `FROM` указывай этот адрес.

Официального образа нет — сначала ищи на Docker Hub образ со значком **Docker Hardened Images**, **Docker Official Images**, **Verified Publisher** или **Sponsored OSS**. Берёшь первый подходящий в этом порядке. Образ без такого значка — только если помеченных нет, и в README одной фразой напиши, чей это образ и почему официального нет.

Чужой форк, зеркало неизвестного автора и образ с тегом `latest` без владельца не используй, пока есть официальный или помеченный вариант.

Если хватает готового образа — не пиши `Dockerfile`. Свой образ собирай поверх базового, выбранного по этому правилу.

## Dockerfile

Свой `Dockerfile` пиши только когда готового образа не хватает. Базовый `FROM` выбирай по разделу «Образы».

Если образ нужен, сборка multistage: зависимости и сборка отдельно, финальный образ без toolchain.

Python-зависимости ставь через `uv`, не через `pip`. Бинарник копируй из официального образа:

```dockerfile
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
```

```dockerfile
FROM python:3.11-slim AS builder
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project

FROM python:3.11-slim
COPY --from=builder /app/.venv /app/.venv
ENV PATH="/app/.venv/bin:$PATH"
```

## Задачи по расписанию

Cron внутри контейнера — [Supercronic](https://github.com/aptible/supercronic), не системный `cron`:

```dockerfile
COPY --from=aptible/supercronic:latest /usr/local/bin/supercronic /usr/local/bin/supercronic
```

Расписание — переменная `CRONTAB` в `.env.example`, не файл в репозитории. `entrypoint.sh` пишет её во временный файл и запускает Supercronic как PID 1:

```sh
#!/bin/sh
set -eu
printf '%s\n' "$CRONTAB" > /tmp/crontab
exec supercronic /tmp/crontab
```

```dotenv
# minute hour day month weekday command
CRONTAB=0 3 * * * /app/backup.sh
TZ=UTC
```

В compose передавай `CRONTAB` и `TZ` через `env_file: .env`. Планировщик долгоживущий: `restart: unless-stopped`. Дефолты `CRONTAB` и `TZ` дублируй в compose, чтобы стек поднялся без `.env`.

## Переменные окружения

В репозиторий коммить только `.env.example`: дефолты и комментарии. `.env` пользователь создаёт сам: `cp .env.example .env`.

Пароли тестовых серверов — известные учебные значения (`postgres` / `postgres`), не секрет боевой машины. Боевые пароли, ключи и токены в репозиторий не клади.

Каждый дефолт из `.env.example`, который читает compose, повтори как `${VAR:-значение}` в `docker-compose.yaml`.

Перед новым хост-портом проверь остальные `.env.example` и compose-файлы.

## README.md

В каждой папке сервиса. Не больше 100 строк. Для `common/` лимит 30 строк, см. «Общие оверлеи». Стиль информационный: коротко, прямо, без канцелярита. Сначала факт, потом вывод, потом действие. Каждое предложение полезно читателю.

Обязательно:

- одна команда запуска; для каждого оверлея — своя одна команда с полным списком `-f`;
- имена образов и ссылка на карточку Docker Hub, например [postgres:17-alpine](https://hub.docker.com/_/postgres);
- что лежит в volume после старта и какой логин открывает базу;
- URL датасетов и зеркал, пока их не больше 10; если больше — id из `DATASET_IDS` и отсылка к `datasets.conf`, без перечисления всех ссылок;
- оценка времени, если загрузка может превысить 180 секунд.

Папка с суффиксом `-min`: в первом абзаце напиши, что сетап минимальный и для продакшена не подходит, и одним предложением почему.

Привилегированные хост-порты (0–1023): как выключить конфликтующую службу через `systemctl` (`systemctl stop nginx`, `systemctl disable nginx`). Команды `service ... stop` и `/etc/init.d/* stop` не приводи.

## Не изобретай велосипед

Задача «добавить тестовую базу» при уже существующем сервере этой СУБД — не новый compose, не новая папка и не оверлей под датасет. Переиспользуй стек. URL датасета храни в переменной, тот же init его скачивает и кладёт в `/docker-entrypoint-initdb.d`. Отдельный init или `.sh` на каждый датасет не пиши.

Пока датасетов не больше 10, список живёт в `.env.example` одной переменной. Несколько штук — через запятую, в том порядке, в каком их применять:

```dotenv
DATASET_URLS=https://example.com/pagila.sql,https://example.com/dellstore.sql
```

Зеркало, если оно есть, ставь в `DATASET_MIRROR_URLS` на той же позиции. Нет зеркала у конкретного датасета — оставляй пустой слот между запятыми. Init сначала берёт URL из `DATASET_URLS`, при ошибке скачивания — URL с той же позиции в `DATASET_MIRROR_URLS`.

Больше 10 датасетов — URL из `.env` убери. Заведи в папке стека `datasets.conf`: одна запись на строку, формат `id-url`. `id` — короткое имя из букв, цифр и `_`, без дефиса. Дефис после `id` отделяет идентификатор от URL, дальше дефисы принадлежат ссылке.

```text
pagila-https://example.com/pagila.sql
dellstore-https://example.com/dellstore.sql https://mirror.example.com/dellstore.sql
```

Второе поле через пробел — зеркало. Его нет — второе поле не пиши.

В `.env` после этого только идентификаторы, не ссылки:

```dotenv
DATASET_IDS=pagila,dellstore
```

Init по `DATASET_IDS` находит строки в `datasets.conf` и качает их. Неизвестный id — ошибка, а не пропуск. Пустой `DATASET_IDS` значит не грузить каталог целиком: сервер поднимается без этих датасетов. В `.env.example` по умолчанию один-два лёгких id, не весь список.

На 11-м датасете перенеси в `datasets.conf` и те, что уже были в `DATASET_URLS`. Два источника правды не держи.

Тот же приём для ClickHouse и любой другой СУБД, у которой стек уже есть. Новая папка — только другая мажорная версия или второй одновременный экземпляр.

## Как выбирать конфигурацию

Делай самый простой стек, которого хватает для учёбы.

Уже есть папка этой СУБД — не поднимай вторую и не заводи оверлей ради датасета. Добавь URL или id в существующий конфиг, см. «Не изобретай велосипед». К уже работающему серверу подключайся по его сети (`pg-net`, `clickhouse-net`).

Чужой датасет подключай ссылкой и тем же init-контейнером. Не копируй файлы в репозиторий.

Прикладные сервисы (Metabase, MLflow и прочие) в этот репозиторий не добавляй. Для них репозиторий dockerfiles: свой сервер БД туда не клади, подключай внешнюю сеть этого репозитория.

Перед новой зависимостью посмотри соседние `docker-compose.yaml`. Если на хосте уже работает подходящий контейнер, проверь его так:

```bash
docker inspect --format='{
  "Cmd": {{json .Config.Cmd}},
  "Env": {{json .Config.Env}},
  "ImageManifestDescriptor": {{json .ImageManifestDescriptor}}
}' <имя_или_id_контейнера>
```

Реальные пароли чужого `.env` в файлы этого репозитория не копируй.

Подключение к чужому PostgreSQL, если оно всё же нужно в этой папке:

- внутреннее имя сети — `db-net`, `external: true`;
- на хосте по умолчанию `pg-net`, переменная `DB_NETWORK_NAME`;
- к `db-net` подключены только сервисы, которым нужна БД;
- хост в `.env.example` — `container_name`, не IP.

```yaml
services:
  dbinit:
    networks:
      - db-net

networks:
  db-net:
    name: ${DB_NETWORK_NAME:-pg-net}
    external: true
```

Другие внешние сервисы (S3, прокси, DNS) — отдельная внешняя сеть провайдера (`minio-net`, `caddymasq`), только у тех сервисов, которым она нужна.

## Проверка образов

После добавления или смены образа в compose или Dockerfile проверь, что он существует: `docker pull <образ>`. Качать до конца не нужно — достаточно, чтобы pull пошёл без ошибки (появились слои).

Образ, который не пуллится, в конфиг не оставляй.
