# Тема 9. Журналы
* __Настройте выполнение контрольной точки раз в 30 секунд.__
```
postgres=# show checkpoint_timeout ;
 5min

postgres=# ALTER SYSTEM SET checkpoint_timeout = '30s';
ALTER SYSTEM

postgres@ubuntu:/etc/postgresql/15/main$ sudo pg_ctlcluster 15 main restart

postgres=# show checkpoint_timeout ;
 checkpoint_timeout 
--------------------
 30s
(1 row)
```
* __10 минут c помощью утилиты pgbench подавайте нагрузку.__
```
postgres@ubuntu:/etc/postgresql/15/main$ pgbench -c8 -P 30 -T 600 -U postgres postgres
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 30.0 s, 936.1 tps, lat 8.512 ms stddev 5.475, 0 failed
progress: 60.0 s, 955.8 tps, lat 8.344 ms stddev 5.477, 0 failed
progress: 90.0 s, 943.7 tps, lat 8.451 ms stddev 5.802, 0 failed
progress: 120.0 s, 891.0 tps, lat 8.947 ms stddev 5.827, 0 failed
progress: 150.0 s, 895.0 tps, lat 8.904 ms stddev 5.470, 0 failed
progress: 180.0 s, 874.6 tps, lat 9.111 ms stddev 5.830, 0 failed
progress: 210.0 s, 766.2 tps, lat 10.403 ms stddev 7.479, 0 failed
progress: 240.0 s, 743.7 tps, lat 10.727 ms stddev 8.358, 0 failed
progress: 270.0 s, 745.9 tps, lat 10.686 ms stddev 7.807, 0 failed
progress: 300.0 s, 769.1 tps, lat 10.374 ms stddev 8.165, 0 failed
progress: 330.0 s, 833.2 tps, lat 9.574 ms stddev 7.705, 0 failed
progress: 360.0 s, 873.2 tps, lat 9.137 ms stddev 6.805, 0 failed
progress: 390.0 s, 930.4 tps, lat 8.576 ms stddev 6.481, 0 failed
progress: 420.0 s, 938.4 tps, lat 8.502 ms stddev 6.345, 0 failed
progress: 450.0 s, 931.4 tps, lat 8.566 ms stddev 6.595, 0 failed
progress: 480.0 s, 938.9 tps, lat 8.498 ms stddev 6.980, 0 failed
progress: 510.0 s, 1015.2 tps, lat 7.860 ms stddev 5.847, 0 failed
progress: 540.0 s, 1004.0 tps, lat 7.948 ms stddev 6.099, 0 failed
progress: 570.0 s, 959.9 tps, lat 8.314 ms stddev 6.763, 0 failed
progress: 600.0 s, 932.3 tps, lat 8.558 ms stddev 7.095, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 536358
number of failed transactions: 0 (0.000%)
latency average = 8.922 ms
latency stddev = 6.669 ms
initial connection time = 16.469 ms
tps = 893.920094 (without initial connection time)
```
* __Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.__
```
postgres@ubuntu:/etc/postgresql/15/main$ cd /var/lib/postgresql/15/main/pg_wal/
postgres@ubuntu:/var/lib/postgresql/15/main/pg_wal$ ls -lhrt
total 4,1G
...
-rw------- 1 postgres postgres  16M июл 10 11:26 000000010000000200000064
-rw------- 1 postgres postgres  16M июл 10 11:27 000000010000000200000066
-rw------- 1 postgres postgres  16M июл 10 11:27 000000010000000200000065
-rw------- 1 postgres postgres  16M июл 10 11:27 000000010000000200000067
-rw------- 1 postgres postgres  16M июл 10 11:28 000000010000000200000068
-rw------- 1 postgres postgres  16M июл 10 11:28 00000001000000020000006A
-rw------- 1 postgres postgres  16M июл 10 11:28 000000010000000200000069
-rw------- 1 postgres postgres  16M июл 10 11:28 00000001000000020000006B
-rw------- 1 postgres postgres  16M июл 10 11:29 00000001000000020000006D
-rw------- 1 postgres postgres  16M июл 10 11:29 00000001000000020000006C
-rw------- 1 postgres postgres  16M июл 10 11:29 00000001000000020000006E
-rw------- 1 postgres postgres  16M июл 10 11:30 00000001000000020000006F
-rw------- 1 postgres postgres  16M июл 10 11:30 000000010000000200000070
-rw------- 1 postgres postgres  16M июл 10 11:30 000000010000000200000072
-rw------- 1 postgres postgres  16M июл 10 11:31 000000010000000200000071
-rw------- 1 postgres postgres  16M июл 10 11:31 000000010000000200000073
-rw------- 1 postgres postgres  16M июл 10 11:31 000000010000000200000075
-rw------- 1 postgres postgres  16M июл 10 11:32 000000010000000200000074
-rw------- 1 postgres postgres  16M июл 10 11:32 000000010000000200000076
-rw------- 1 postgres postgres  16M июл 10 11:32 000000010000000200000078
-rw------- 1 postgres postgres  16M июл 10 11:33 000000010000000200000077
-rw------- 1 postgres postgres  16M июл 10 11:33 000000010000000200000079
-rw------- 1 postgres postgres  16M июл 10 11:33 00000001000000020000007A
-rw------- 1 postgres postgres  16M июл 10 11:33 00000001000000020000007B
-rw------- 1 postgres postgres  16M июл 10 11:34 00000001000000020000007C
-rw------- 1 postgres postgres  16M июл 10 11:34 00000001000000020000007D
-rw------- 1 postgres postgres  16M июл 10 11:34 00000001000000020000007E
-rw------- 1 postgres postgres  16M июл 10 11:35 00000001000000020000007F
-rw------- 1 postgres postgres  16M июл 10 11:35 000000010000000200000080
-rw------- 1 postgres postgres  16M июл 10 11:35 000000010000000200000081
-rw------- 1 postgres postgres  16M июл 10 11:35 000000010000000200000082
-rw------- 1 postgres postgres  16M июл 10 11:36 000000010000000200000083
-rw------- 1 postgres postgres  16M июл 10 11:36 000000010000000200000084
-rw------- 1 postgres postgres  16M июл 10 11:36 000000010000000200000085
-rw------- 1 postgres postgres  16M июл 10 11:38 000000010000000100000086
```
Сгенерировалось 35 файлов по 16 Мб => 560 Мб

560 Мб (за 10 мин) / 30 сек (checkpoint_timeout) = 18,6 Мб на 1  контрольную точку.

* __Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?__
```
postgres@ubuntu:/var/lib/postgresql/15/main/global$ sudo /usr/lib/postgresql/15/bin/pg_controldata /var/lib/postgresql/15/main/
pg_control version number:            1300
Catalog version number:               202209061
Database system identifier:           7249060859101800008
Database cluster state:               in production
pg_control last modified:             Пн 10 июл 2023 11:37:50
Latest checkpoint location:           1/862D2AC8
Latest checkpoint's REDO location:    1/862D2A90
Latest checkpoint's REDO WAL file:    000000010000000100000086
Latest checkpoint's TimeLineID:       1
Latest checkpoint's PrevTimeLineID:   1
Latest checkpoint's full_page_writes: on
Latest checkpoint's NextXID:          0:705509
Latest checkpoint's NextOID:          41450
Latest checkpoint's NextMultiXactId:  1
Latest checkpoint's NextMultiOffset:  0
Latest checkpoint's oldestXID:        716
Latest checkpoint's oldestXID's DB:   1
Latest checkpoint's oldestActiveXID:  705509
Latest checkpoint's oldestMultiXid:   1
Latest checkpoint's oldestMulti's DB: 5
Latest checkpoint's oldestCommitTsXid:0
Latest checkpoint's newestCommitTsXid:0
Time of latest checkpoint:            Пн 10 июл 2023 11:37:46
Fake LSN counter for unlogged rels:   0/3E8
Minimum recovery ending location:     0/0
Min recovery ending loc's timeline:   0
Backup start location:                0/0
Backup end location:                  0/0
End-of-backup record required:        no
wal_level setting:                    replica
wal_log_hints setting:                off
max_connections setting:              40
max_worker_processes setting:         8
max_wal_senders setting:              10
max_prepared_xacts setting:           0
max_locks_per_xact setting:           64
track_commit_timestamp setting:       off
Maximum data alignment:               8
Database block size:                  8192
Blocks per segment of large relation: 131072
WAL block size:                       8192
Bytes per WAL segment:                16777216
Maximum length of identifiers:        64
Maximum columns in an index:          32
Maximum size of a TOAST chunk:        1996
Size of a large-object chunk:         2048
Date/time type storage:               64-bit integers
Float8 argument passing:              by value
Data page checksum version:           0
Mock authentication nonce:            5c16ad02302388c871fd47a895f95ad488222fe23f68db1174325cb5cfb53622

postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 1/862D2B78
(1 row)


postgres=# SELECT pg_walfile_name('1/862D2B78');
     pg_walfile_name      
--------------------------
 000000010000000100000086
(1 row)

postgres=# CHECKPOINT;
CHECKPOINT

postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 1/862D2C60
(1 row)

postgres@ubuntu:/var/lib/postgresql/15/main/global$ sudo /usr/lib/postgresql/15/bin/pg_waldump -p /var/lib/postgresql/15/main/pg_wal -s 1/862D2B78 -e 1/862D2C60 000000010000000100000086

rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 1/862D2B78, prev 1/862D2B40, desc: RUNNING_XACTS nextXid 705509 latestCompletedXid 705508 oldestRunningXid 705509

rmgr: XLOG        len (rec/tot):    114/   114, tx:          0, lsn: 1/862D2BB0, prev 1/862D2B78, desc: CHECKPOINT_ONLINE redo 1/862D2B78; tli 1; prev tli 1; fpw true; xid 0:705509; oid 41450; multi 1; offset 0; oldest xid 716 in DB 1; oldest multi 1 in DB 5; oldest/newest commit timestamp xid: 0/0; oldest running xid 705509; online

rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 1/862D2C28, prev 1/862D2BB0, desc: RUNNING_XACTS nextXid 705509 latestCompletedXid 705508 oldestRunningXid 705509
```
В журнал попадает запись о том, что контрольная точка пройдена - CHECKPOINT_ONLINE.
```
postgres@ubuntu:/var/lib/postgresql/15/main/global$ sudo /usr/lib/postgresql/15/bin/pg_controldata /var/lib/postgresql/15/main/ | egrep 'Latest.*location'
Latest checkpoint location:           1/862D2BB0
Latest checkpoint's REDO location:    1/862D2B78
```
Общий объем можно оценить как(1 + checkpoint_completion_target) * объем-между-контр-точками.
Таким образом большая часть контрольных точек происходитпо расписанию, раз в checkpoint_timeout единиц времени. 
Но при повышенной нагрузке контрольная точка вызывается чаще, чтобы постараться уложиться в объем max_wal_size

* __Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.__

_tps в синхронном режиме:_
```
postgres=# show commit_delay ;
 commit_delay 
--------------
 0
(1 row)

postgres=# show commit_siblings ;
 commit_siblings 
-----------------
 5
(1 row)

postgres=# show synchronous_commit ;
 synchronous_commit 
--------------------
 on
(1 row)

postgres@ubuntu:~$ pgbench -c8 -P 30 -T 120 -U postgres postgres
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 30.0 s, 1015.0 tps, lat 7.854 ms stddev 5.311, 0 failed
progress: 60.0 s, 1002.5 tps, lat 7.956 ms stddev 5.487, 0 failed
progress: 90.0 s, 1029.6 tps, lat 7.740 ms stddev 4.996, 0 failed
progress: 120.0 s, 982.3 tps, lat 8.112 ms stddev 5.267, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 120 s
number of transactions actually processed: 120891
number of failed transactions: 0 (0.000%)
latency average = 7.914 ms
latency stddev = 5.269 ms
initial connection time = 14.482 ms
tps = 1007.330773 (without initial connection time)
```
_tps в асинхронном режиме:_
```
alter system set synchronous_commit = 'off';
alter system set wal_writer_delay = '200ms';

postgres@ubuntu:~$ sudo pg_ctlcluster 15 main restart

postgres=# show synchronous_commit ;
 synchronous_commit 
-------------------
 off
(1 row)

postgres=# show wal_writer_delay;
 wal_writer_delay 
------------------
 200ms
(1 row)

postgres@ubuntu:~$ pgbench -c8 -P 30 -T 120 -U postgres postgres
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 30.0 s, 4426.7 tps, lat 1.770 ms stddev 1.547, 0 failed
progress: 60.0 s, 4376.6 tps, lat 1.800 ms stddev 1.323, 0 failed
progress: 90.0 s, 4100.5 tps, lat 1.916 ms stddev 1.538, 0 failed
progress: 120.0 s, 3834.9 tps, lat 2.054 ms stddev 1.594, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 120 s
number of transactions actually processed: 502169
number of failed transactions: 0 (0.000%)
latency average = 1.879 ms
latency stddev = 1.506 ms
initial connection time = 13.871 ms
tps = 4184.164496 (without initial connection time)
```
tps в синхронном режиме составило 1007, в асинхронном 4184
__tps = 1007.330773 (without initial connection time)__
__tps = 4184.164496 (without initial connection time)__

Пропускная способность (число транзакций в секунду, подсчитанное без учёта времени установления подключения к серверу) в асинхронном режиме больше, т.к. во время этого режима транзакции завершаются быстрее (но увеличивается вероятность потери данных при падении бд) 

В синхронном режиме сервер ждет сохранения записей WAL транзакции в постоянном хранилище, поэтому tps намного меньше. 

* __Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?__
```
pg_createcluster --datadir=/var/lib/postgresql/15/summ 15 summ --  --data-checksums

postgres@ubuntu:~$ pg_createcluster --datadir=/var/lib/postgresql/15/summ 15 summ --  --data-checksums
Creating new PostgreSQL cluster 15/summ ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/summ --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.
The database cluster will be initialized with this locale configuration:
  provider:    libc
  LC_COLLATE:  en_US.UTF-8
  LC_CTYPE:    en_US.UTF-8
  LC_MESSAGES: en_US.UTF-8
  LC_MONETARY: ru_RU.UTF-8
  LC_NUMERIC:  ru_RU.UTF-8
  LC_TIME:     ru_RU.UTF-8
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".
Data page checksums are enabled.
fixing permissions on existing directory /var/lib/postgresql/15/summ ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Europe/Moscow
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Warning: systemd does not know about the new cluster yet. Operations like "service postgresql start" will not handle it. To fix, run:
  sudo systemctl daemon-reload
Ver Cluster Port Status Owner    Data directory              Log file
15  summ    5434 down   postgres /var/lib/postgresql/15/summ /var/log/postgresql/postgresql-15-summ.log

postgres@ubuntu:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
13  main    5433 online postgres /var/lib/postgresql/13/main /var/log/postgresql/postgresql-13-main.log
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
15  summ    5434 down   postgres /var/lib/postgresql/15/summ /var/log/postgresql/postgresql-15-summ.log

postgres@ubuntu:~$ sudo pg_ctlcluster 15 summ start
postgres@ubuntu:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
13  main    5433 online postgres /var/lib/postgresql/13/main /var/log/postgresql/postgresql-13-main.log
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
15  summ    5434 online postgres /var/lib/postgresql/15/summ /var/log/postgresql/postgresql-15-summ.log

postgres@ubuntu:~$ psql -p 5434

postgres=# CREATE TABLE people(
  id serial,
  fio char(100)
); 
CREATE TABLE

postgres=# INSERT INTO people(fio) SELECT 'noname' FROM generate_series(1,100);
INSERT 0 100

postgres@ubuntu:~$ sudo pg_ctlcluster 15 summ stop

postgres=# SELECT pg_relation_filepath('people');

 pg_relation_filepath 
----------------------
 base/5/16389
(1 row)

postgres@ubuntu:~$ dd if=/dev/zero of=/var/lib/postgresql/15/summ/base/5/16389 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0,0101595 s, 0,8 kB/s

postgres@ubuntu:~$ sudo pg_ctlcluster 15 summ start
postgres@ubuntu:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
13  main    5433 online postgres /var/lib/postgresql/13/main /var/log/postgresql/postgresql-13-main.log
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
15  summ    5434 online postgres /var/lib/postgresql/15/summ /var/log/postgresql/postgresql-15-summ.log

postgres@ubuntu:~$ psql -p 5434

postgres=# select * from people;
WARNING:  page verification failed, calculated checksum 44039 but expected 31982
ERROR:  invalid page in block 0 of relation base/5/16389
postgres=# 
```
Ошибка контрольной суммы, изменим параметр ignore_checksum_failure , чтобы проигнорировать ошибку и продолжить работу
```
postgres=# alter system set ignore_checksum_failure='on';
ALTER SYSTEM

postgres=# show ignore_checksum_failure;
 ignore_checksum_failure 
-------------------------
 off
(1 row)

postgres@ubuntu:~$ sudo pg_ctlcluster 15 summ restart

postgres@ubuntu:~$ psql -p 5434

postgres=# select * from people limit 10;
 id |                                                 fio                                                  
----+------------------------------------------------------------------------------------------------------
  1 | noname                                                                                              
  2 | noname                                                                                              
  3 | noname                                                                                              
  4 | noname                                                                                              
  5 | noname                                                                                              
  6 | noname                                                                                              
  7 | noname                                                                                              
  8 | noname                                                                                              
  9 | noname                                                                                              
 10 | noname                                                                                              
(10 rows)
```