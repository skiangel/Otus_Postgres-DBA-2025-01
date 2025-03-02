# Настройка PostgreSQL #

В задаче к домашнему заданию предлагается сначала настроить сервер на максимальную производительность, однако, Мы сначала прогоним pgbench на дефолтных настройках:
Создадим 

```postgresql
CREATE DATABASE test;
```

Запустим инициализацию базы и тест:

```shell
pgbench -h 127.0.0.1 -U denis -i test
Password:
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.05 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 1.60 s (drop tables 0.00 s, create tables 0.16 s, client-side generate 0.77 s, vacuum 0.15 s, primary keys 0.52 s).
```

```shell
 pgbench -h 127.0.0.1 -U  -c 50 -j 2 -P 10 -T 60 test
Password:
pgbench (16.6 (Ubuntu 16.6-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 10.0 s, 74.4 tps, lat 582.504 ms stddev 614.266, 0 failed
progress: 20.0 s, 83.2 tps, lat 595.483 ms stddev 746.335, 0 failed
progress: 30.0 s, 85.0 tps, lat 577.095 ms stddev 876.744, 0 failed
progress: 40.0 s, 84.4 tps, lat 615.133 ms stddev 832.168, 0 failed
progress: 50.0 s, 85.3 tps, lat 590.667 ms stddev 673.886, 0 failed
progress: 60.0 s, 83.8 tps, lat 567.585 ms stddev 766.017, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 5011
number of failed transactions: 0 (0.000%)
latency average = 595.470 ms
latency stddev = 776.416 ms
initial connection time = 598.884 ms
tps = 83.584422 (without initial connection time)
```

Видим TPS: **tps = 83.584422**

Теперь поправим настройки сервера. На сервере у нас 2 ядра и 8 гигабайт оперативной памяти.

запишем в файл **/etc/postgresql/16/main/conf.d/max.conf** следующие настройки:

````shell
shared_buffers = 3200MB
effective_cache_size = 6100MB
work_mem = 128MB
maintenance_work_mem = 1024MB
checkpoint_timeout = 30min
````

и запустим pgbench:

````shell
pgbench -h 127.0.0.1 -U denis -c 50 -j 2 -P 10 -T 60 test
Password:
pgbench (16.6 (Ubuntu 16.6-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 10.0 s, 79.2 tps, lat 536.533 ms stddev 630.567, 0 failed
progress: 20.0 s, 85.3 tps, lat 586.634 ms stddev 770.716, 0 failed
progress: 30.0 s, 86.1 tps, lat 607.429 ms stddev 638.830, 0 failed
progress: 40.0 s, 86.6 tps, lat 559.111 ms stddev 677.111, 0 failed
progress: 50.0 s, 86.8 tps, lat 576.721 ms stddev 845.581, 0 failed
progress: 60.0 s, 87.0 tps, lat 559.751 ms stddev 719.445, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 5160
number of failed transactions: 0 (0.000%)
latency average = 578.740 ms
latency stddev = 727.311 ms
initial connection time = 570.896 ms
tps = 85.999807 (without initial connection time)
````

Видим достаточно незначительное увеличение производительности:
**tps = 85.999807** против начальных **tps = 83.584422**

Возьмем настройки с сайта pgtune:
```shell
max_connections = 200
shared_buffers = 2GB
effective_cache_size = 6GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 5242kB
huge_pages = off
min_wal_size = 1GB
max_wal_size = 4GB
```

```shell
 sudo systemctl restart postgresql
denis@edupostgres01:~$ pgbench -h 127.0.0.1 -U denis -c 50 -j 2 -P 10 -T 60 test
Password:
pgbench (16.6 (Ubuntu 16.6-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 10.0 s, 78.9 tps, lat 554.632 ms stddev 649.888, 0 failed
progress: 20.0 s, 86.5 tps, lat 578.575 ms stddev 738.662, 0 failed
progress: 30.0 s, 85.9 tps, lat 581.534 ms stddev 773.793, 0 failed
progress: 40.0 s, 85.8 tps, lat 579.908 ms stddev 753.852, 0 failed
progress: 50.0 s, 87.1 tps, lat 586.724 ms stddev 793.762, 0 failed
progress: 60.0 s, 85.2 tps, lat 582.962 ms stddev 580.231, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 5144
number of failed transactions: 0 (0.000%)
latency average = 580.567 ms
latency stddev = 719.355 ms
initial connection time = 552.520 ms
tps = 85.694814 (without initial connection time)
```

Видимо, на данной конфигурации мы достигли предела быстродействия.


## Дополнительное задание 
Теперь воспользуемся утилитой **sysbench-tpcc**, для этого сначала установим **sysbench**
Установка родительской утилиты описана на гитхабе и не вызывает сложностей.
**sysbench-tpcc** устанавливаем путем клонирования с git

```shell
git clone https://github.com/Percona-Lab/sysbench-tpcc.git
```

Запустим инициализацию базы данных для тестов

```shell
cd sysbench-tpcc
./tpcc.lua --pgsql-host=127.0.0.1 --pgsql-user=denis --pgsql-password=some_hard_password --pgsql-db=test --time=300 --threads=64 --report-interval=1 --tables=10 --scale=100 --db-driver=pgsql prepare
```

Подготовка базы падает в segmentation fault  и после этого кластер отказывается принимать соединения с сообщением 
```shell
psql: error: connection to server at "127.0.0.1", port 5432 failed: FATAL:  the database system is not yet accepting connections
DETAIL:  Consistent recovery state has not been yet reached.
```

Подозреваю, что указал слишком много потоков.
К счастью, у меня есть еще один кластер, попробуем работать с ним, указав всего два потока:
````shell
 ./tpcc.lua --pgsql-host=127.0.0.1 --pgsql-port=5433 --pgsql-user=denis --pgsql-password=some_strong_password --pgsql-db=test --time=300 --threads=2 --report-interval=1 --tables=10 --scale=100 --db-driver=pgsql prepare
````

Пока идет подготовка базы, заглянул в лог сервера, и увидел, что мое подозрение насчет слишком большого количества потоков не подтвердилось.
Оказалось, что банально кончилось место на разделе, где у меня находится data_directory кластера main.
```shell
2025-03-01 13:24:48.800 UTC [89456] FATAL:  could not write to file "pg_wal/xlogtemp.89456": No space left on device
```

Учитывая, что это у меня виртуалка, просто делаю на хосте

```shell
sudo virsh blockresize edu_postgres01 /var/lib/libvirt/images/edu_postgres01.qcow2 50G
```
Указывая путь до образа нужного диска.
Затем, уже на виртуалке:
```shell
sudo growpart /dev/vdb 1
sudo resize2fs /dev/vdb1
```

Готово:
```shell
df -H
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              833M  1,3M  832M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv  103G   49G   50G  50% /
tmpfs                              4,2G  1,1M  4,2G   1% /dev/shm
tmpfs                              5,3M     0  5,3M   0% /run/lock
/dev/vda2                          2,1G  194M  1,8G  11% /boot
/dev/vdb1                           53G   10G   41G  20% /mnt/data
tmpfs                              833M   13k  833M   1% /run/user/1000
```

Я прервал подготовку БД во втором кластере, Восстановил первый и запустил подготовку на нем, уменьшив параметры tables и scale, чтобоы уменьшить размер базы
```shell
 ./tpcc.lua --pgsql-host=127.0.0.1 --pgsql-port=5433 --pgsql-user=denis --pgsql-password=some_strong_password --pgsql-db=test --time=300 --threads=64 --report-interval=1 --tables=3 --scale=25 --db-driver=pgsql prepare
```

После чего запустил бенчмарк:

```shell
./tpcc.lua --pgsql-host=127.0.0.1 --pgsql-port=5433 --pgsql-user=denis --pgsql-password=some_strong_password --pgsql-db=test --time=300 --threads=64 --report-interval=1 --tables=3 --scale=25 --db-driver=pgsql run
```

Получили результат
```shell
SQL statistics:
    queries performed:
        read:                            39813
        write:                           41350
        other:                           6744
        total:                           87907
    transactions:                        2980   (9.83 per sec.)
    queries:                             87907  (290.02 per sec.)
    ignored errors:                      343    (1.13 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          303.1008s
    total number of events:              2980

Latency (ms):
         min:                                    1.53
         avg:                                 6491.72
         max:                                46525.82
         95th percentile:                    19425.29
         sum:                             19345324.34

Threads fairness:
    events (avg/stddev):           46.5625/5.28
    execution time (avg/stddev):   302.2707/0.56
```

Это результат с модифицированными настройками. уберем файл из  /etc/postgresql/16/main/conf.d/ и снова запустим тест
```shell
sudo mv  /etc/postgresql/16/main/conf.d/* /home/denis
 ./tpcc.lua --pgsql-host=127.0.0.1 --pgsql-port=5433 --pgsql-user=denis --pgsql-password=some_strong_pasword --pgsql-db=test --time=300 --threads=64 --report-interval=1 --tables=3 --scale=25 --db-driver=pgsql run
```

Результат отличается, но не сильно:

```shell
SQL statistics:
    queries performed:
        read:                            37719
        write:                           39038
        other:                           6450
        total:                           83207
    transactions:                        2836   (9.20 per sec.)
    queries:                             83207  (269.78 per sec.)
    ignored errors:                      333    (1.08 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          308.4219s
    total number of events:              2836

Latency (ms):
         min:                                    3.43
         avg:                                 6889.55
         max:                                35837.26
         95th percentile:                    18075.36
         sum:                             19538759.56

Threads fairness:
    events (avg/stddev):           44.3125/4.62
    execution time (avg/stddev):   305.2931/2.91
```