# Настройка Zabbix

[Zabbix](https://www.zabbix.com/) - система мониторинга состояния сервера. Данная система является полностью независимой от сторонних ресурсов, т.к. серверная часть (отвечает за обработку метрик), агент (отвечает за сбор метрик) и фронт (отвечает за отображение метрик) располагаются на сервере приложения (или на отдельно вынесенном сервере).

---

***./docker-compose.yml:***

```yaml
version: '3.3'
services:
  ...other services...
  
  zabbix-server:
    image: zabbix/zabbix-server-pgsql:ubuntu-5.2-latest
    depends_on:
      - db
    ports:
      - 10051:10051     # порт для принятия активных метрик от агента
    env_file:
      - ./zabbix/.env

  zabbix-agent:
    image: zabbix/zabbix-agent:ubuntu-5.2-latest
    ports:
      - 10050:10050 # порт для принятия пассивных метрик от сервера

  zabbix-frontend:
    image: zabbix/zabbix-web-nginx-pgsql:ubuntu-5.2-latest
    depends_on:
      - db
      - zabbix-server
    env_file:
      - ./zabbix/.env
    ports:
      - 80:8080

  ...more other services...
```

***./db/.env:***

```yaml
POSTGRES_DB=postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres

ZABBIX_USER=zabbix_user               # имя и пароль Zabbix-пользователя
ZABBIX_PASSWORD=zabbix_user_password  # в БД для хранения метрик

```

***./db/Dockerfile:***

```Dockerfile
FROM postgres:12.4-alpine

RUN mkdir ./pg_logs             # создание директории
RUN chmod -R 777 /pg_logs       # для лог-файла
RUN chown -R postgres /pg_logs  # с необходимыми правами

COPY ./init.sh /docker-entrypoint-initdb.d  # копирование инициализационного для Zabbix`а скрипта
COPY ./postgresql.conf /etc/postgresql/postgresql.conf  # копирование кастомного конфиг-файла для PostgreSQL
```

***./db/init.sh:***

```bash
#!/bin/bash

psql -c "
    CREATE USER $ZABBIX_USER WITH PASSWORD '$ZABBIX_PASSWORD' INHERIT;  # создание Zabbix-пользователя
    GRANT pg_monitor TO $ZABBIX_USER;                                   # в БД
"

```

***./zabbix/agent/.env:***

```yaml
ZBX_SERVER_HOST=zabbix-server   # hostname Zabbix-сервера
```

***./zabbix/.env:***

```yaml
DB_SERVER_HOST=db   # hostname БД
POSTGRES_USER=zabbix_user                 # имя и пароль
POSTGRES_PASSWORD=zabbix_user_password    # Zabbix-пользователя в БД
POSTGRES_DB=postgres  # имя БД для хранения метрик
```

Ссылки:

1. [Zabbix-сервер](https://hub.docker.com/r/zabbix/zabbix-server-pgsql);
2. [Zabbix-агент](https://hub.docker.com/r/zabbix/zabbix-agent);
3. [Zabbix-агент](https://hub.docker.com/r/zabbix/zabbix-agent);
