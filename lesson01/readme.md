# SQL и реляционные СУБД. Введение в PostgreSQL 

## Установка Postgresql сервера

Мы будем устанвливать сервер на виртуальную машину, 
работающую под управлением гипервизора KVM. Установка гипервизора и ВМ являются рутинными и здесь рассматриваться не 
будут.

На виртуальной машине установлена Ubuntu 24.04.01 LTS
Обновим репозитории и установим сервер:
```shell
apt update
apt install postgresql-16
```

## Работа с разными уровнями изоляции транзакций

Запустим две psql, для этого переключимся в пользователя postgres, так как мы не знаем его пароль в СУБД:

```shell
sudo -u postgres -s
psql
```

Создаем таблицу и наполняем ее данными:

```postgresql
 create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
```

проверяем уровень изоляции, в обеих сессиях он одинаковый:

```postgresql
postgres=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)
```

в первой сессии начнем транзакцию, вставляем данные, но не делаем commit
```postgresql
BEGIN;
 insert into persons(first_name, second_name) values('sergey', 'sergeev');
```

Во второй сессии выбираем все из таблицы persons:

```postgresql
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```

Новой записи не видим, так как не завершили транзакцию в первой сессии, при этом у нас уровень изоляции 
**read committed**, который не позволяет получать данные незавершенных транзакций

После завершения транзакции в первой сессии, во второй мы можем видеть новую запись.

Начнем новые сессии с уровнем изоляции транзакций repeatable read
В первой сессии добавляем данные в таблицу, но не заканчиваем транзакцию.
Во второй сессии выбираем данные из таблицы и не видим изменений, произведенных первой сессией.
Заканчиваем транзакцию в первой сессии, но изменений, внесенных ей не видим во второй сессии, так как в ней
начата транзакция с уровнем изоляции  repeatable read.
Заканчиваем транзакцию во второй сессии и только после этого можем в ней увидеть 
изменения, внесенные в первой сессии:

```postgresql
postgres=*# commit;
COMMIT
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

