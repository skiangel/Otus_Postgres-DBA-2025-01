# Логический уровень PostgreSQL

Создадим новый кластер:

```shell
sudo pg_createcluster 16 second
sudo pg_ctlcluster 16 second start
pg_lsclusters

Ver Cluster Port Status Owner    Data directory                Log file
16  main    5432 online postgres /mnt/data/16/main             /var/log/postgresql/postgresql-16-main.log
16  second  5433 online postgres /var/lib/postgresql/16/second /var/log/postgresql/postgresql-16-second.log
```

Подключимся к кластеру и проверим, что подключились к нужному

```shell
sudo -u postgres psql -p 5433
psql (16.6 (Ubuntu 16.6-0ubuntu0.24.04.1))
Type "help" for help.

postgres=# SELECT name, setting FROM pg_settings WHERE name IN ('data_directory', 'port');
      name      |            setting
----------------+-------------------------------
 data_directory | /var/lib/postgresql/16/second
 port           | 5433
(2 rows)
```

Создадим новую базу данных, в ней cхему таблицу и занесем в нее значение:

```shell
postgres=# create database testdb;
CREATE DATABASE
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# create schema testnm;
CREATE SCHEMA
testdb=# create table testnm.t1 (c1 int);
CREATE TABLE
testdb=# INSERT INTO testnm.t1 VALUES (1);
INSERT 0 1
```

Мы указываем перед именем таблицы имя схемы, иначе таблица создастся в схеме public

Создадим новую роль, дадим ей право на соединение с базой, право использования созданой схемы и право select из всех таблиц схемы

```shell
testdb=# create role readonly with nologin;
CREATE ROLE
testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT
testdb=# GRANT USAGE ON SCHEMA testnm to readonly;
GRANT
testdb=# GRANT SELECT on all TABLES in SCHEMA testnm TO readonly;
GRANT
testdb=# ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly;
ALTER DEFAULT PRIVILEGES
```

Последней командой мы изменяем дефолтное поведение разрешений в схеме, благодаря этому вновь создаваемые таблицы будут доступны для чтения роли readonly в этой схеме.

Создадим пользователя и назначим ему роль readonly

```shell
testdb=# CREATE USER testread with password 'test123';
CREATE ROLE
testdb=# grant readonly to testread;
GRANT ROLE
```

Подключимся к базе под именем созданного пользователя:

```shell
psql -h 127.0.0.1 -p 5433 -U testread -d testdb
Password for user testread:
psql (16.6 (Ubuntu 16.6-0ubuntu0.24.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

testdb=>
```

При попытке получить данные без указания схемы получим ошибку, так как прав на схему public мы не давали:
```shell
testdb=> select * from t1;
ERROR:  permission denied for table t1
testdb=> select * from testnm.t1;
 c1
----
  1
  2
(2 rows)
```

При явном указании схемы все проходит нормально.

Так как мы предусмотрели расположение таблицы в нужной схеме, а так же выдачу прав на вновь создаваемые таблицы, то мы счастливо избежали проблем, писаных в пунктах 20-36 домашнего задания. 

Попытаемся создать новую таблицу:
```shell
testdb=> create table t2(c1 integer);
ERROR:  permission denied for schema public
LINE 1: create table t2(c1 integer);
```

В домашнем задании предполагалось, что у нас получится, однако мы получили ошибку.
Все дело в том, что в ДЗ предполагается работа с версией Postgresql 14 в котором права create и usage на схеме public были у всех.
У нас же установлена версия 16, а начиная с версии 15 права create в public есть только у владельца базы.

в версиях ранее 15 можно было забрать права с помощью REVOKE:

```shell
REVOKE CREATE on SCHEMA public FROM public;
```
