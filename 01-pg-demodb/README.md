# 01-pg-demodb

Учебный PostgreSQL с демобазой «Авиаперевозки» Postgres Pro: https://postgrespro.ru/education/demodb

```bash
docker compose up -d
```

Образ — [postgres:18.6](https://hub.docker.com/_/postgres), последняя стабильная ветка официального образа. Init `busybox:1.37` скачивает дамп в volume `init-sql`. Сервер при пустом каталоге данных применяет его из `/docker-entrypoint-initdb.d`.

После старта в volume `postgres-data` лежит кластер PostgreSQL 18. База `demo`, схема `bookings`. Снаружи: `localhost:5432`, логин `postgres` / `postgres`. Другие compose-проекты ходят в контейнер `pg` по сети `pg-net`.

Датасет: https://edu.postgrespro.ru/demo-20250901-2y.sql.gz (полёты за два года, сжатый файл около 1,1 ГБ, база около 11 ГБ). Зеркала нет.

Первый запуск занимает десятки минут: скачивание около 1,1 ГБ и загрузка базы около 11 ГБ. На проверке это заняло примерно 16 минут. Таймаут деплоя doco-cd по умолчанию 180 секунд, его нужно поднять.

У сервера задан `max_wal_size=4GB`. Без этого во время загрузки чекпоинты идут каждые несколько секунд.

## Версия сервера

Postgres Pro указывает минимум PostgreSQL 15: https://postgrespro.ru/education/demodb

В схеме функции `bookings.lang`, `bookings.now` и `bookings.version` записаны стандартным SQL-телом (`LANGUAGE sql ... RETURN ...`), а не строкой в `$$`. Такой синтаксис есть с PostgreSQL 14. Первичный ключ `routes` — ограничение-исключение GiST (`route_no WITH =`, `validity WITH &&`); темпоральный ключ `WITHOUT OVERLAPS` в дампе не используется, он появится в PostgreSQL 18. Отдельной конструкции ровно 15-й версии в схеме нет: порог 15 зафиксировали авторы демобазы.

Файл `demo-20250901-2y.sql.gz` снят `pg_dump` 17.5 и содержит `SET transaction_timeout = 0`. Параметр `transaction_timeout` появился в PostgreSQL 17, поэтому этот скрипт на 15 и 16 останавливается с ошибкой. Образ 18.6 выше заявленного минимума и принимает дамп без правок.

Скрипт начинается с `DROP DATABASE demo`. На пустом кластере такой базы ещё нет, и при `ON_ERROR_STOP` загрузка оборвалась бы. Init кладёт `00-prepare.sh`, который создаёт пустую `demo` до дампа.
