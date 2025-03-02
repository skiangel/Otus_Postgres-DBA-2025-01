# MVCC, vacuum и autovacuum

У нас уже есть виртуальнам машина с установленным postgresql 16, воспользуемся ей.
Создадим базу данных для тестов:

```postgresql
CREATE DATABASE testvacuum;
CREATE DATABASE
```

Проинициализируем ее с pgbench:
```shell
pgbench -i testvacuum -h 127.0.0.1 -U denis
```

Запустим тесты:
```shell
pgbench -c8 -P 6 -T 60 -U denis testvacuum -h 127.0.0.1
```

Получаем результат:
```shell
pgbench (16.6 (Ubuntu 16.6-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 6.0 s, 81.2 tps, lat 93.749 ms stddev 64.138, 0 failed
progress: 12.0 s, 84.0 tps, lat 95.225 ms stddev 70.762, 0 failed
progress: 18.0 s, 86.5 tps, lat 92.321 ms stddev 58.922, 0 failed
progress: 24.0 s, 85.7 tps, lat 93.213 ms stddev 56.050, 0 failed
progress: 30.0 s, 88.5 tps, lat 90.719 ms stddev 64.129, 0 failed
progress: 36.0 s, 85.8 tps, lat 93.030 ms stddev 68.152, 0 failed
progress: 42.0 s, 85.2 tps, lat 93.911 ms stddev 65.007, 0 failed
progress: 48.0 s, 88.3 tps, lat 90.242 ms stddev 67.128, 0 failed
progress: 54.0 s, 86.2 tps, lat 92.922 ms stddev 66.470, 0 failed
progress: 60.0 s, 86.8 tps, lat 91.578 ms stddev 61.507, 0 failed
transaction type: <builtin: TPC-B (sort of)>
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 5157
number of failed transactions: 0 (0.000%)
latency average = 92.758 ms
latency stddev = 64.434 ms
initial connection time = 235.816 ms
tps = 86.172623 (without initial connection time)
```

Поместим в файл  **/etc/postgresql/16/main/conf.d/vacuumtest.conf** настройки, прикреплённые к ДЗ и перезапустим сервер.
Запустим бенчмарк снова.

Результат: 
```shell
 pgbench -c8 -P 6 -T 60 -U denis testvacuum -h 127.0.0.1
Password:
pgbench (16.6 (Ubuntu 16.6-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 6.0 s, 82.5 tps, lat 91.218 ms stddev 58.223, 0 failed
progress: 12.0 s, 87.7 tps, lat 91.551 ms stddev 57.924, 0 failed
progress: 18.0 s, 85.2 tps, lat 91.668 ms stddev 61.960, 0 failed
progress: 24.0 s, 88.2 tps, lat 92.471 ms stddev 67.289, 0 failed
progress: 30.0 s, 88.5 tps, lat 90.597 ms stddev 60.827, 0 failed
progress: 36.0 s, 85.5 tps, lat 93.868 ms stddev 69.484, 0 failed
progress: 42.0 s, 89.3 tps, lat 89.446 ms stddev 57.235, 0 failed
progress: 48.0 s, 83.0 tps, lat 96.591 ms stddev 72.325, 0 failed
progress: 54.0 s, 87.7 tps, lat 90.880 ms stddev 64.446, 0 failed
progress: 60.0 s, 86.3 tps, lat 91.910 ms stddev 61.935, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 5191
number of failed transactions: 0 (0.000%)
latency average = 92.102 ms
latency stddev = 63.409 ms
initial connection time = 272.977 ms
tps = 86.785598 (without initial connection time)
```
Особой разницы нет. Я так думаю, что это результат крайне скромных системных ресурсов виртуальной машины.

Создадим таблицу для тестов:

```postgresql
CREATE TABLE moretest (name text);
```

Заполним ее одним миллионом записей:

```postgresql
INSERT INTO moretest(name) SELECT 'noname' FROM generate_series(1,1000000);
```

Посмотрим размер таблицы:
```postgresql
SELECT pg_size_pretty(pg_table_size('moretest'));
 pg_size_pretty
----------------
 35 MB
(1 row)
```

Пять раз обновим значения в таблице:

```postgresql
testvacuum=# update moretest set name = 'name1';
UPDATE 1000000
testvacuum=# update moretest set name = 'name2';
UPDATE 1000000
testvacuum=# update moretest set name = 'name3';
UPDATE 1000000
testvacuum=# update moretest set name = 'name4';
UPDATE 1000000
testvacuum=# update moretest set name = 'name5';
UPDATE 1000000
```

Смотрим количество мертвых записей:

```postgresql
testvacuum=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'moretest';
 relname  | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
----------+------------+------------+--------+-------------------------------
 moretest |    1000000 |    4998662 |    299 | 2025-03-02 12:25:00.613137+00

```

Фактически пять миллионов мертвых записей, ведь при апдейте создается новая запись, а старая помечается как мертвая.

Еще раз смотрим количество мертвых заиписей и видим, что пришел автовакуум и все почистил:

```postgresql
 SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'moretest';
 relname  | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
----------+------------+------------+--------+-------------------------------
 moretest |    1000000 |          0 |      0 | 2025-03-02 12:25:00.613137+00
(1 row)
```

Смотрим размер файла таблицы:

```postgresql
 SELECT pg_size_pretty(pg_table_size('moretest'));
 pg_size_pretty
----------------
 173 MB
(1 row)
```

Размер файла вырос, это объясняется тем, что автовакуум не отдает свободное место операционной система, а лишь помечает его как свободное внутри файла.

Отключаем автовакуум на нашей таблице:
```postgresql
ALTER TABLE moretest SET (autovacuum_enabled = off);
ALTER TABLE
```

Десять раз обновим таблицу, добавляя к значениям символ

```postgresql
testvacuum=# update moretest set name = 'name1';
UPDATE 1000000
testvacuum=# update moretest set name = 'name12';
UPDATE 1000000
testvacuum=# update moretest set name = 'name123';
UPDATE 1000000
testvacuum=# update moretest set name = 'name1234';
UPDATE 1000000
testvacuum=# update moretest set name = 'name12345';
UPDATE 1000000
testvacuum=# update moretest set name = 'name123456';
UPDATE 1000000
testvacuum=# update moretest set name = 'name1234567';
UPDATE 1000000
testvacuum=# update moretest set name = 'name12345678';
UPDATE 1000000
testvacuum=# update moretest set name = 'name123456789';
UPDATE 1000000
testvacuum=# update moretest set name = 'name0123456789';
UPDATE 1000000
```
Смотрим размер фала таблицы:
```postgresql
testvacuum=# SELECT pg_size_pretty(pg_table_size('moretest'));
 pg_size_pretty
----------------
 434 MB
(1 row)
```

Такой размер можно объяснить, посмотрев количество мертвых строк:

```postgresql
testvacuum=#  SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'moretest';
 relname  | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
----------+------------+------------+--------+-------------------------------
 moretest |    1000000 |    9997139 |    999 | 2025-03-02 12:33:01.587933+00
(1 row)
```

Фактически 10 миллионов мертвых строк, ведь мы выключили автовакуум и помеченные мертвыми строки никем не чистятся.

Радикально решим вопрос, произведя vacuum full:
```postgresql
testvacuum=# vacuum full moretest;
VACUUM
testvacuum=#  SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'moretest';
 relname  | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
----------+------------+------------+--------+-------------------------------
 moretest |    1000000 |    9997139 |    999 | 2025-03-02 12:33:01.587933+00
(1 row)

testvacuum=# SELECT pg_size_pretty(pg_table_size('moretest'));
 pg_size_pretty
----------------
 42 MB
(1 row)
```

Размер приблизился к первоначальному.

## Задание во звездочкой
```postgresql
DO $$
DECLARE
BEGIN
FOR one IN 1 .. 10 BY 1 LOOP
update moretest set name = one;
RAISE NOTICE 'Итерация %', one;
end loop;
end$$;
```