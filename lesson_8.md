# MVCC, vacuum и autovacuum.
* __Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB__   
_Инстанс создан_

* __Установить на него PostgreSQL 15 с дефолтными настройками__   
_PostgreSQL 15 установлен_
```
postgres@ubuntu:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
* __Создать БД для тестов: выполнить pgbench -i postgres__
```
postgres@ubuntu:~$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.10 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.38 s (drop tables 0.01 s, create tables 0.03 s, client-side generate 0.17 s, vacuum 0.08 s, primary keys 0.09 s).
```
* __Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres__
```
postgres@ubuntu:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 658.8 tps, lat 11.984 ms stddev 4.720, 0 failed
progress: 12.0 s, 667.0 tps, lat 11.900 ms stddev 4.856, 0 failed
progress: 18.0 s, 714.0 tps, lat 11.118 ms stddev 4.797, 0 failed
progress: 24.0 s, 634.0 tps, lat 12.523 ms stddev 5.713, 0 failed
progress: 30.0 s, 724.8 tps, lat 10.949 ms stddev 4.789, 0 failed
progress: 36.0 s, 713.8 tps, lat 11.131 ms stddev 5.573, 0 failed
progress: 42.0 s, 687.4 tps, lat 11.541 ms stddev 4.982, 0 failed
progress: 48.0 s, 650.9 tps, lat 12.187 ms stddev 5.719, 0 failed
progress: 54.0 s, 675.6 tps, lat 11.746 ms stddev 5.505, 0 failed
progress: 60.0 s, 617.6 tps, lat 12.833 ms stddev 6.490, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 40472
number of failed transactions: 0 (0.000%)
latency average = 11.764 ms
latency stddev = 5.362 ms
initial connection time = 23.773 ms
tps = 674.352888 (without initial connection time)
```
* __Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла__   
_Изначальные параметры:_
```
max_connections = 100                   # (change requires restart)
shared_buffers = 128MB                  # min 128kB
#effective_cache_size = 4GB
#maintenance_work_mem = 64MB            # min 1MB
#checkpoint_completion_target = 0.9     # checkpoint target duration, 0.0 - 1.0
#wal_buffers = -1                       # min 32kB, -1 sets based on shared_buffers
#default_statistics_target = 100        # range 1-10000
#random_page_cost = 4.0                 # same scale as above
#effective_io_concurrency = 1           # 1-1000; 0 disables prefetching
#work_mem = 4MB                         # min 64kB
min_wal_size = 80MB
max_wal_size = 1GB
```
_В файле /etc/postgresql/15/main/postgresql.conf изменены параметры на новые значения:_
```
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```
_и перезагружен кластер_
```
sudo pg_ctlcluster 15 main stop
sudo pg_ctlcluster 15 main start
```
* __Протестировать заново__
```
postgres@ubuntu:/etc/postgresql/15/main$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 965.6 tps, lat 8.231 ms stddev 5.137, 0 failed
progress: 12.0 s, 990.3 tps, lat 8.052 ms stddev 4.825, 0 failed
progress: 18.0 s, 948.3 tps, lat 8.393 ms stddev 4.684, 0 failed
progress: 24.0 s, 954.1 tps, lat 8.357 ms stddev 4.986, 0 failed
progress: 30.0 s, 1008.7 tps, lat 7.903 ms stddev 4.633, 0 failed
progress: 36.0 s, 987.0 tps, lat 8.083 ms stddev 5.899, 0 failed
progress: 42.0 s, 1004.7 tps, lat 7.944 ms stddev 5.794, 0 failed
progress: 48.0 s, 962.8 tps, lat 8.283 ms stddev 6.945, 0 failed
progress: 54.0 s, 896.3 tps, lat 8.901 ms stddev 6.589, 0 failed
progress: 60.0 s, 946.0 tps, lat 8.433 ms stddev 6.165, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 57990
number of failed transactions: 0 (0.000%)
latency average = 8.251 ms
latency stddev = 5.620 ms
initial connection time = 16.689 ms
tps = 966.099745 (without initial connection time)
```
* __Что изменилось и почему?__

_За 1 минуту работы с 8 одновременными соединениями кластер обработал 57990 транзакций._ 
_tps - пропускная способность (число транзакций в секунду, подсчитанное без учёта времени установления подключения к серверу) увеличилось с 674 до 966 => пропускная способность базы увеличилась после изменения параметров._ 

_Что могло повлиять:_ 
_Изменение параметра shared_buffers с 128MB до 1GB позволило увеличить  наше доступное пространство кэша._

_Изменение параметра max_wal_size с 1 до 16 GB, что позволяет снизить нагрузку, возникающую вследствие более частого сброса «грязных» страниц данных на диск._

_Изменение параметра min_wal_size с 80Mb до 4GB ограничивает снизу число файлов WAL, которые будут переработаны для будущего использования; такой объём WAL всегда будет перерабатываться, даже если система простаивает и оценка использования говорит, что нужен совсем небольшой WAL._

_Установлен параметр checkpoint_completion_target на 0.9, который задаёт целевое время для завершения процедуры контрольной точки, как коэффициент для общего времени между контрольными точками. Используется, чтобы избежать «заваливания» системы ввода/вывода при резкой интенсивной записи страниц._

_Установлен параметр effective_io_concurrency на 2 - задаёт допустимое число параллельных операций ввода/вывода, которое говорит PostgreSQL о том, сколько операций ввода/вывода могут быть выполнены одновременно. Чем больше это число, тем больше операций ввода/вывода будет пытаться выполнить параллельно PostgreSQL в отдельном сеансе._

_Установлен параметр maintenance_work_mem на 512MB - задаёт максимальный объём памяти для операций обслуживания БД, в частности VACUUM, CREATE INDEX и ALTER TABLE ADD FOREIGN KEY._

_Изменение параметра max_connections со 100 до 40 никак не повлияло, т.к. мы запускали тесты всего с 8 одновременными соединениями._

* __Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк__
```
postgres=# CREATE TABLE people(
  id serial,
  fio char(100)
); 
CREATE TABLE

postgres=# INSERT INTO people(fio) SELECT 'noname' FROM generate_series(1,1000000);
INSERT 0 1000000
```
* __Посмотреть размер файла с таблицей__
```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('people'));
 pg_size_pretty 
----------------
 135 MB
(1 row)
```
* __5 раз обновить все строчки и добавить к каждой строчке любой символ__
```
postgres=# update people set fio = 'name';
UPDATE 1000000

postgres=# update people set fio = 'name2';
UPDATE 1000000

postgres=# update people set fio = 'name3';
UPDATE 1000000

postgres=# update people set fio = 'name4';
UPDATE 1000000

postgres=# update people set fio = 'name5';
UPDATE 1000000
```
* __Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум__
```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'people';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 people  |    1000000 |    2000000 |    199 | 2023-06-30 16:16:45.372244+03
(1 row)
```
* __Подождать некоторое время, проверяя, пришел ли автовакуум__
```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'people';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 people  |    1000000 |          0 |      0 | 2023-06-30 16:17:47.857015+03
```
* __5 раз обновить все строчки и добавить к каждой строчке любой символ__
```
postgres=# update people set fio = 'name11';
UPDATE 1000000

postgres=# update people set fio = 'name22';
UPDATE 1000000

postgres=# update people set fio = 'name33';
UPDATE 1000000

postgres=# update people set fio = 'name44';
UPDATE 1000000

postgres=# update people set fio = 'name55';
UPDATE 1000000

postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'people';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 people  |    1000000 |    4999760 |    499 | 2023-06-30 16:17:47.857015+03
(1 row)

postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'people';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 people  |    1000000 |          0 |      0 | 2023-06-30 16:19:48.177349+03
(1 row)
```
* __Посмотреть размер файла с таблицей__
```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('people'));
 pg_size_pretty 
----------------
 808 MB
(1 row)
```
* __Отключить Автовакуум на конкретной таблице__
```
postgres=# ALTER TABLE people set (autovacuum_enabled = off);
ALTER TABLE
```
* __10 раз обновить все строчки и добавить к каждой строчке любой символ__
```
postgres=# update people set fio = 'name111';
UPDATE 1000000

postgres=# update people set fio = 'name222';
UPDATE 1000000

postgres=# update people set fio = 'name333';
UPDATE 1000000

postgres=# update people set fio = 'name444';
UPDATE 1000000

postgres=# update people set fio = 'name555';
UPDATE 1000000

postgres=# update people set fio = 'name666';
UPDATE 1000000

postgres=# update people set fio = 'name777';
UPDATE 1000000

postgres=# update people set fio = 'name888';
UPDATE 1000000

postgres=# update people set fio = 'name999';
UPDATE 1000000

postgres=# update people set fio = 'name101010';
UPDATE 1000000
```
* __Посмотреть размер файла с таблицей__
```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('people'));

 pg_size_pretty 
----------------
 1482 MB
(1 row)
```
* __Объясните полученный результат__

_Т.к. мы отключили автовакуум, очистка мертвых строк прекратилась и на данный момент количество мертвех строк = 9998447. Именно они занимают место в таблице._
```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'people';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 people  |    1000000 |    9998447 |    999 | 2023-06-30 16:19:48.177349+03
(1 row)
```
_При запуске автовакуума мертвые строки очищаются, однако размер таблицы не уменьшается:_
```
postgres=# ALTER TABLE people set (autovacuum_enabled = on);
ALTER TABLE

postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'people';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 people  |    1813885 |          0 |      0 | 2023-06-30 16:34:44.117895+03
(1 row)

postgres=# SELECT pg_size_pretty(pg_total_relation_size('people'));
 pg_size_pretty 
----------------
 1482 MB
(1 row)
```
_Для уменьшения размера таблицы необходимо выполнить vacuum full_
```
postgres=# vacuum full;
VACUUM

postgres=# SELECT pg_size_pretty(pg_total_relation_size('people'));
 pg_size_pretty 
----------------
 135 MB
(1 row)
```
