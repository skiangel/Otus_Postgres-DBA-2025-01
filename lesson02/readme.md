# Установка PostgreSQL 

## Виртуальные машины
- Мы установили две виртуальные машины для запуска контейнеров с сервером и клиентом.
- Машины установлены на гипервизоре KVM

## Установка Docker Engine на виртуальные машины

Будем устанавливать Docker Engine из официального apt репозитория, для этого воспользуемся инструкцией на [официальном сайте](https://docs.docker.com/engine/install/ubuntu/).

### Подключаем репозиторий

```shell
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

### Устанавливаем собственно Docker 

```shell
 sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Создание и запуск контейнера с postgresql и подключение к нему

Для запуска контейнера используем скрипт

```shell
#!/bin/bash
docker run -d \
    --name some-postgres \
    -e POSTGRES_PASSWORD=mypassword \
    -e PGDATA=/var/lib/postgresql/data/pgdata \
    -v /var/lib/postgres:/var/lib/postgresql/data \
    -p 5432:5432 \
    postgres
```
С файловой системы хоста мы смонтировали директорию /var/lib/postgres внутрь контейнера к директории /var/lib/postgres/data
Забегая вперед, поясню, что это делается для того, чтобы данные сохранялись при удалении контейнера.

Запустим контейнер с клиентом postgresql

```shell
sudo docker run -it --rm jbergknoff/postgresql-client postgresql://postgres:mypassword@10.73.200.38:5432/postgres
```

Подключимся к серверу postgresql, запущенному в контейнере:

```shell
psql -h 10.73.200.38 -U postgres -d postgres
```
Создадим таблицу и вставим в нее несколько строк:
```postgresql
CREATE TABLE users ( Id SERIAL PRIMARY KEY, FirstName CHARACTER VARYING(30));
INSERT INTO users (FirstName) VALUES ('Denis');
INSERT INTO users (FirstName) VALUES ('Alexander');
INSERT INTO users (FirstName) VALUES ('Natasha');
```

проверим, что данные попали в базу

```postgresql
 select * from users;
 id | firstname
----+-----------
  1 | Denis
  2 | Alexander
  3 | Natashka
(3 rows)
```

Остановим и удалим контейнер с сервером, после чего создадим и запустим его снова:

```shell
 sudo docker ps
CONTAINER ID   IMAGE      COMMAND                  CREATED          STATUS          PORTS                                       NAMES
676ed4860268   postgres   "docker-entrypoint.s…"   45 minutes ago   Up 45 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   some-postgres

sudo docker stop 676ed4860268
676ed4860268
sudo docker remove 676ed4860268
sudo docker run -d \
    --name some-postgres \
    -e POSTGRES_PASSWORD=mypassword \
    -e PGDATA=/var/lib/postgresql/data/pgdata \
    -v /var/lib/postgres:/var/lib/postgresql/data \
    -p 5432:5432 \
    postgres
```

Подключимся к серверу с локальной машины, на которую предвалительно поставили postgresql-client:

```shell
 psql -h 10.73.200.38 -U postgres -d postgres
Password for user postgres:
psql (16.6 (Ubuntu 16.6-0ubuntu0.24.04.1), server 17.3 (Debian 17.3-1.pgdg120+1))
WARNING: psql major version 16, server major version 17.
         Some psql features might not work.
Type "help" for help.
```

Посмотрим, сохранились ли нажи изменения в базе

```postgresql
 SELECT * FROM users;
 id | firstname
----+-----------
  1 | Denis
  2 | Alexander
  3 | Natashka
(3 rows)
```

Все на месте, так как namespace находится на файловой системе хоста, которую мы примонтировали внутрь контейнера.








