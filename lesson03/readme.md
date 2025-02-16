# Физический уровень PostgreSQL

## Виртуальные машины

Мы будем использовать виртуальные машины, созданные в предыдущем уроке
Postgresql на них уже установлен.

Проверяем, запущен ли кластер:

```shell
denis@edupostgres01:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

На второй виртуальной машине ситуация такая же.

## Работа с базами данных

Создадим базу данных test01, в ней создадим таблицу test и добавим в нее несоклько строк:

```postgresql
CREATE DATABASE test01;
\c test01
CREATE TABLE test(c1 text);
insert into test values('1');
insert into test values('2');
\q
```

Остановим кластер:

```shell
sudo systemctl status postgresql@16-main
```

В материалах к домашнему заданию предлагается остановить кластер через команду **sudo -u postgres pg_ctlcluster 15 main stop**
Однако, это вызвает системное предупреждение
```shell
sudo -u postgres pg_ctlcluster 16 main stop
''Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@16-main
```

С последующим переходом сервиса в состояние **failed** вместо **inactive**:
```shell
denis@edupostgres01:~$ sudo systemctl status postgresql@16-main
× postgresql@16-main.service - PostgreSQL Cluster 16-main
     Loaded: loaded (/usr/lib/systemd/system/postgresql@.service; enabled-runtime; preset: enabled)
     Active: failed (Result: exit-code) since Sun 2025-02-16 11:09:49 UTC; 2min 9s ago
   Duration: 2d 5h 3min 24.627s
    Process: 60335 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 16-main start (code=exited, status=0/SUC>
    Process: 73454 ExecStop=/usr/bin/pg_ctlcluster --skip-systemctl-redirect -m fast 16-main stop (code=exited, status>
   Main PID: 60341 (code=exited, status=0/SUCCESS)
        CPU: 1min 24.975s
```

На хосте виртуализации добавим еще один диск вирутальной машины:
```shell
sudo qemu-img create -f qcow2 -o preallocation=off /var/lib/libvirt/images_second/edu_postgres01_2.qcow2 10G
Formatting '/var/lib/libvirt/images_second/edu_postgres01_2.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=off preallocation=off compression_type=zlib size=10737418240 lazy_refcounts=off refcount_bits=16
```

Присоединим новый диск к вируальной машине:
```shell
sudo virsh attach-disk edu_postgres01 /var/lib/libvirt/images_second/edu_postgres01_2.qcow2 vdb --cache none
Disk attached successfully
```
Создадим на диске раздел, отформатируем его и примонтируем к **/mnt/data**

```shell
sudo fdisk /dev/vdb
[sudo] password for denis:

Welcome to fdisk (util-linux 2.39.3).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier 0x9fe86390.

Command (m for help): g
Created a new GPT disklabel (GUID: 422C2564-55B2-46EE-AC97-D72DBBC2EC5F).

Command (m for help): n
Partition number (1-128, default 1):
First sector (34-351, default 34): 34
Last sector, +/-sectors or +/-size{K,M,G,T,P} (34-351, default 350):

Created a new partition 1 of type 'Linux filesystem' and of size 158,5 KiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

sudo mkfs.ext4 /dev/vdb1
mke2fs 1.47.0 (5-Feb-2023)
Discarding device blocks: done
Creating filesystem with 2620928 4k blocks and 655360 inodes
Filesystem UUID: de5ef12b-3092-461d-a5ce-a15ddace132d
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
sudo mkdir /mnt/data
sudo mount /dev/vda1 /mnt/data
```

Добавим запись в /etc/fstab:

```shell
/dev/vdb1 /mnt/data ext4 defaults 0 1
```

Перезагрузим виртуальную машину и проверим, что диск остался примонтированным:

```shell
 df -H
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              833M  1,2M  832M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv  103G  7,3G   91G   8% /
tmpfs                              4,2G  1,1M  4,2G   1% /dev/shm
tmpfs                              5,3M     0  5,3M   0% /run/lock
/dev/vdb1                           11G   25k   10G   1% /mnt/data
/dev/vda2                          2,1G  100M  1,9G   6% /boot
tmpfs                              833M   13k  833M   1% /run/user/1000
```

Остановим кластер и перенесем содержимое /var/lib/postgres/16 в /mnt/data

```shell
sudo systemctl stop postgresql@16-main.service
sudo mv /var/lib/postgresql/16 /mnt/data
```

Если сейчас попробовать запустить кластер, то получим ошибку:
```shell
 sudo systemctl start postgresql@16-main.service
Job for postgresql@16-main.service failed because the service did not take the steps required by its unit configuration.
See "systemctl status postgresql@16-main.service" and "journalctl -xeu postgresql@16-main.service" for details.
```

Для ее устранения надо указать в конфигурационном файле postgres новое расположение каталога **DATA**

```shell
echo "data_directory = '/mnt/data/16/main'" >> /etc/postgresql/postgresql.conf
```

Запустим кластер и проверим его состояние:

```shell
 sudo systemctl start postgresql@16-main.service
denis@edupostgres01:~$ sudo systemctl status postgresql@16-main.service
● postgresql@16-main.service - PostgreSQL Cluster 16-main
     Loaded: loaded (/usr/lib/systemd/system/postgresql@.service; enabled-runtime; preset: enabled)
     Active: active (running) since Sun 2025-02-16 12:05:08 UTC; 10s ago
    Process: 1388 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 16-main start (code=exited, sta>
   Main PID: 1393 (postgres)
      Tasks: 6 (limit: 9445)
     Memory: 18.5M (peak: 27.1M)
        CPU: 350ms
     CGroup: /system.slice/system-postgresql.slice/postgresql@16-main.service
             ├─1393 /usr/lib/postgresql/16/bin/postgres -D /mnt/data/16/main -c config_file=/etc/postgresql/>
             ├─1394 "postgres: 16/main: checkpointer "
             ├─1395 "postgres: 16/main: background writer "
             ├─1397 "postgres: 16/main: walwriter "
             ├─1398 "postgres: 16/main: autovacuum launcher "
             └─1399 "postgres: 16/main: logical replication launcher "

фев 16 12:05:05 edupostgres01 systemd[1]: Starting postgresql@16-main.service - PostgreSQL Cluster 16-main...
фев 16 12:05:08 edupostgres01 systemd[1]: Started postgresql@16-main.service - PostgreSQL Cluster 16-main.
```
Подсоединимся к базе и проверим, остались ли наши изменения:

```postgresql
psql
 \c test01
You are now connected to database "test01" as user "postgres".
test01=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | test | table | postgres
(1 row)
SELECT * FROM test;
 c1
----
 1
 2
(2 rows)
```

Все в порядке, кластер запустился, данные на месте.


Остановм кластер, отсоединим диск и присоединим его ко второй виртуальной машине.
После этого на второй машине остановми кластер, очистим директорию data и примонтируем к ней диск от первой виртуальной машины:
```shell
sudo systemctl start postgresql@16-main.service
sudo rm -r /var/lib/postgresql/*
sudo mount /dev/vdb1 /var/lib/postgresql/
sudo systemctl start postgresql@16-main.service
psql
```

Проверим, на месте ли наши данные:

```postgresql
postgres=# \l
                                                       List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | ICU Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+-------------+-------------+------------+-----------+-----------------------
 postgres  | postgres | UTF8     | libc            | ru_RU.UTF-8 | ru_RU.UTF-8 |            |           |
 template0 | postgres | UTF8     | libc            | ru_RU.UTF-8 | ru_RU.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | ru_RU.UTF-8 | ru_RU.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
 test01    | postgres | UTF8     | libc            | ru_RU.UTF-8 | ru_RU.UTF-8 |            |           |
(4 rows)

postgres=# \c test01
You are now connected to database "test01" as user "postgres".
test01=# SELECT * FROM test;
 c1
----
 1
 2
(2 rows)
```

Все нормально, диск перенесся, кластер видит данные.
