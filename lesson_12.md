# Урок 12. Настройка PostgreSQL 
• развернуть виртуальную машину любым удобным способом
• поставить на неё PostgreSQL 15 любым способом

### Параметры ВМ:
DB Version: 15
OS Type: linux
DB Type: mixed
Total Memory (RAM): 4 GB
CPUs num: 2
Connections num: 100
Data Storage: ssd

__Установлен postgres 15 версии:__
```
postgres@ubuntu:~$ psql -p 5434
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
```
__Изначальные параметры:__
```
show max_connections;
show shared_buffers;
show effective_cache_size;
show maintenance_work_mem;
show checkpoint_completion_target;
show wal_buffers;
show default_statistics_target;
show random_page_cost;
show effective_io_concurrency;
show work_mem;
show min_wal_size;
show max_wal_size;

 max_connections 
-----------------
 100

 shared_buffers 
----------------
 128MB

 effective_cache_size 
----------------------
 4GB

 maintenance_work_mem 
----------------------
 64MB

 checkpoint_completion_target 
------------------------------
 0.9

 wal_buffers 
-------------
 4MB

 default_statistics_target 
---------------------------
 100

 random_page_cost 
------------------
 4

 effective_io_concurrency 
--------------------------
 1

 work_mem 
----------
 4MB

 min_wal_size 
--------------
 80MB

 max_wal_size 
--------------
 1GB
```
• настроить кластер PostgreSQL 15 на максимальную производительность не
обращая внимание на возможные проблемы с надежностью в случае
аварийной перезагрузки виртуальной машины

__Добавлена бд postgres для нагрузочного тестирования__
```
postgres@ubuntu:~$ pgbench -i postgres -p 5434
```
## PG_TUNE
__Установлены новые значения параметров, предложенные pg_tune:__
```
ALTER SYSTEM SET max_connections = '100';
ALTER SYSTEM SET shared_buffers = '1GB';
ALTER SYSTEM SET effective_cache_size = '3GB';
ALTER SYSTEM SET maintenance_work_mem = '256MB';
ALTER SYSTEM SET checkpoint_completion_target = '0.9';
ALTER SYSTEM SET wal_buffers = '4MB';
ALTER SYSTEM SET default_statistics_target = '100';
ALTER SYSTEM SET random_page_cost = '4';
ALTER SYSTEM SET effective_io_concurrency = '1';
ALTER SYSTEM SET work_mem = '4MB';
ALTER SYSTEM SET min_wal_size = '80MB';
ALTER SYSTEM SET max_wal_size = '1GB';
```
_Перезагрузка кластера:_
```
postgres@ubuntu:~$ pg_ctlcluster 15 summ stop
postgres@ubuntu:~$ pg_ctlcluster 15 summ start
```
• нагрузить кластер через утилиту через утилиту pgbench (https://postgrespro.ru/docs/postgrespro/14/pgbench)
• написать какого значения tps удалось достичь, показать какие параметры в
какие значения устанавливали и почему
__Выполнено нагрузочное тестирование:__
```
postgres@ubuntu:~$ pgbench -c8 -P 10 -T 120 -U postgres -p 5434 postgres
...
number of transactions actually processed: 26212
number of failed transactions: 0 (0.000%)
latency average = 36.573 ms
latency stddev = 32.115 ms
initial connection time = 12.013 ms
tps = 218.390176 (without initial connection time)
```
## PGCONFIGURATOR
_Возвращены параметры обратно:_
```
ALTER SYSTEM SET max_connections = '100';
ALTER SYSTEM SET shared_buffers = '128MB';
ALTER SYSTEM SET effective_cache_size = '4GB';
ALTER SYSTEM SET maintenance_work_mem = '64MB';
ALTER SYSTEM SET checkpoint_completion_target = '0.9';
ALTER SYSTEM SET wal_buffers = '16MB';
ALTER SYSTEM SET default_statistics_target = '100';
ALTER SYSTEM SET random_page_cost = '1.1';
ALTER SYSTEM SET effective_io_concurrency = '200';
ALTER SYSTEM SET work_mem = '2621kB';
ALTER SYSTEM SET min_wal_size = '1GB';
ALTER SYSTEM SET max_wal_size = '4GB';
```
__И выставлены новые исходя из рекомендаций pgconfigurator:__
```
# Connectivity
ALTER SYSTEM SET max_connections = '100';
ALTER SYSTEM SET superuser_reserved_connections = '3';

# Memory Settings
ALTER SYSTEM SET shared_buffers = '1024 MB';
ALTER SYSTEM SET work_mem = '32 MB';
ALTER SYSTEM SET maintenance_work_mem = '320 MB';
ALTER SYSTEM SET huge_pages = 'off';
ALTER SYSTEM SET effective_cache_size = '3 GB';
ALTER SYSTEM SET effective_io_concurrency = '100';
ALTER SYSTEM SET random_page_cost = '1.25';

# Checkpointing:
ALTER SYSTEM SET checkpoint_timeout = '15 min';
ALTER SYSTEM SET checkpoint_completion_target = '0.9';
ALTER SYSTEM SET max_wal_size = '1024 MB';
ALTER SYSTEM SET min_wal_size = '512 MB';


# WAL writing
ALTER SYSTEM SET wal_compression = 'on';
ALTER SYSTEM SET wal_buffers = '-1';
ALTER SYSTEM SET wal_writer_delay = '200ms';
ALTER SYSTEM SET wal_writer_flush_after = '1MB';


# Background writer
ALTER SYSTEM SET bgwriter_delay = '200ms';
ALTER SYSTEM SET bgwriter_lru_maxpages = '100';
ALTER SYSTEM SET bgwriter_lru_multiplier = '2.0';
ALTER SYSTEM SET bgwriter_flush_after = '0';

# Parallel queries:
ALTER SYSTEM SET max_worker_processes = '2';
ALTER SYSTEM SET max_parallel_workers_per_gather = '1';
ALTER SYSTEM SET max_parallel_maintenance_workers = '1';
ALTER SYSTEM SET max_parallel_workers = '2';
ALTER SYSTEM SET parallel_leader_participation = 'on';

# Advanced features
ALTER SYSTEM SET enable_partitionwise_join = 'on';
ALTER SYSTEM SET enable_partitionwise_aggregate = 'on';
ALTER SYSTEM SET jit = 'on';
ALTER SYSTEM SET max_slot_wal_keep_size = '1000 MB';
ALTER SYSTEM SET track_wal_io_timing = 'on';
ALTER SYSTEM SET maintenance_io_concurrency = '100';
ALTER SYSTEM SET wal_recycle = 'on';
```
_Перезагрузка кластера:_
```
postgres@ubuntu:~$ pg_ctlcluster 15 summ stop
postgres@ubuntu:~$ pg_ctlcluster 15 summ start
postgres@ubuntu:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  summ    5434 online postgres /var/lib/postgresql/15/summ /var/log/postgresql/postgresql-15-summ.log
```
__Нагрузочное тестирование__
```
postgres@ubuntu:~$ pgbench -c8 -P 10 -T 120 -U postgres -p 5434 postgres
...
number of transactions actually processed: 27175
number of failed transactions: 0 (0.000%)
latency average = 35.280 ms
latency stddev = 29.670 ms
initial connection time = 13.426 ms
tps = 226.404732 (without initial connection time)
```
## PG_TUNE с увеличенными параметрами
__Заданы увеличенные параметры в pg_tune для более мощного сервера:__
DB Version: 15
OS Type: linux
DB Type: mixed
Total Memory (RAM): 12 GB
CPUs num: 6
Connections num: 300
Data Storage: ssd
```
ALTER SYSTEM SET max_connections = '300';
ALTER SYSTEM SET shared_buffers = '3GB';
ALTER SYSTEM SET effective_cache_size = '9GB';
ALTER SYSTEM SET maintenance_work_mem = '768MB';
ALTER SYSTEM SET checkpoint_completion_target = '0.9';
ALTER SYSTEM SET wal_buffers = '16MB';
ALTER SYSTEM SET default_statistics_target = '100';
ALTER SYSTEM SET random_page_cost = '1.1';
ALTER SYSTEM SET effective_io_concurrency = '200';
ALTER SYSTEM SET work_mem = '1747kB';
ALTER SYSTEM SET min_wal_size = '1GB';
ALTER SYSTEM SET max_wal_size = '4GB';
ALTER SYSTEM SET max_worker_processes = '6';
ALTER SYSTEM SET max_parallel_workers_per_gather = '3';
ALTER SYSTEM SET max_parallel_workers = '6';
ALTER SYSTEM SET max_parallel_maintenance_workers = '3';
```
_И проведено тестирование:_
```
postgres@ubuntu:~$ pgbench -c8 -P 10 -T 120 -U postgres -p 5434 postgres
...
number of transactions actually processed: 26929
number of failed transactions: 0 (0.000%)
latency average = 35.606 ms
latency stddev = 31.312 ms
initial connection time = 14.152 ms
tps = 224.354182 (without initial connection time)
```
Однако tps особо не изменился. 
## Ручная настройка
__Установлены большие значения у некоторых параметров с комментарием почему был выбран этот параметр:__
```
ALTER SYSTEM SET max_connections = '300';
ALTER SYSTEM SET shared_buffers = '12GB';
```
Замечено, что в производственных средах большое значение для shared_buffer действительно дает хорошую производительность, хотя для достижения правильного баланса всегда следует проводить тесты.
```
ALTER SYSTEM SET wal_buffers = '1GB';
```
PostgreSQL сначала записывает записи в WAL (журнал предзаписи) в буферы, а затем эти буферы сбрасываются на диск. Размер буфера по умолчанию, определенный wal_buffers, составляет 16 МБ. Но если у вас много одновременных подключений, то более высокое значение может повысить производительность.
```
ALTER SYSTEM SET effective_cache_size = '20GB';
```
effective_cache_size предоставляет оценку памяти, доступной для кэширования диска. Это всего лишь ориентир, а не точный объем выделенной памяти или кеша. Он не выделяет фактическую память, но сообщает оптимизатору объем кеша, доступный в ядре. Если значение этого параметра установлено слишком низким, планировщик запросов может принять решение не использовать некоторые индексы, даже если они будут полезны. Поэтому установка большого значения всегда имеет смысл.
```
ALTER SYSTEM SET work_mem = '128MB';
```
Эта настройка используется для сложной сортировки. Если вам нужно выполнить сложную сортировку, увеличьте значение work_mem для получения хороших результатов. Сортировка в памяти происходит намного быстрее, чем сортировка данных на диске. Установка очень высокого значения может стать причиной узкого места в памяти для вашей среды, поскольку этот параметр относится к операции сортировки пользователя. Поэтому, если у вас много пользователей, пытающихся выполнить операции сортировки, тогда система выделит:
```
ALTER SYSTEM SET maintenance_work_mem = '1GB';
```
maintenance_work_mem — это параметр памяти, используемый для задач обслуживания. Значение по умолчанию составляет 64 МБ. Установка большого значения помогает в таких задачах, как VACUUM, RESTORE, CREATE INDEX, ADD FOREIGN KEY и ALTER TABLE.
```
ALTER SYSTEM SET checkpoint_completion_target = '0.1';
```
checkpoint_completion_target — это доля времени между контрольными точками для завершения контрольной точки. Высокая частота контрольных точек может повлиять на производительность. Для плавного выполнения задания контрольной точки, checkpoint_timeout должен иметь низкое значение. В противном случае ОС будет накапливать все грязные страницы до тех пор, пока соотношение не будет соблюдено, а затем производить большой сброс.
```
ALTER SYSTEM SET checkpoint_timeout = '30 min';
```
Параметр checkpoint_timeout используется для установки времени между контрольными точками WAL. Установка слишком низкого значения уменьшает время восстановления после сбоя, поскольку на диск записывается больше данных, но это также снижает производительность, поскольку каждая контрольная точка в конечном итоге потребляет ценные системные ресурсы.
checkpoint_completion_target — это доля времени между контрольными точками для завершения контрольной точки. Высокая частота контрольных точек может повлиять на производительность. Для плавного выполнения задания контрольной точки, checkpoint_timeout должен иметь низкое значение. В противном случае ОС будет накапливать все грязные страницы до тех пор, пока соотношение не будет соблюдено, а затем производить большой сброс.

_После изменения параметров не удалось стартануть кластер:_
```
postgres@ubuntu:~$ pg_ctlcluster 15 summ stop
postgres@ubuntu:~$ pg_ctlcluster 15 summ start

Error: /usr/lib/postgresql/15/bin/pg_ctl /usr/lib/postgresql/15/bin/pg_ctl start -D /var/lib/postgresql/15/summ -l /var/log/postgresql/postgresql-15-summ.log -s -o  -c config_file="/etc/postgresql/15/summ/postgresql.conf"  exited with status 1: 
2023-08-01 12:18:03.278 MSK [3424] FATAL:  could not map anonymous shared memory: Cannot allocate memory
2023-08-01 12:18:03.278 MSK [3424] HINT:  This error usually means that PostgreSQL's request for a shared memory segment exceeded available memory, swap space, or huge pages. To reduce the request size (currently 14255955968 bytes), reduce PostgreSQL's shared memory usage, perhaps by reducing shared_buffers or max_connections.
2023-08-01 12:18:03.278 MSK [3424] LOG:  database system is shut down
pg_ctl: could not start server
Examine the log output.
```
_Уменьшен параметр shared_buffers на '3GB', кластер запустился, после чего запущено нагрузочное тестирование_
```
postgres@ubuntu:/var/lib/postgresql/15/summ$ pgbench -c8 -P 10 -T 120 -U postgres -p 5434 postgres
...
number of transactions actually processed: 27920
number of failed transactions: 0 (0.000%)
latency average = 34.338 ms
latency stddev = 29.747 ms
initial connection time = 13.758 ms
tps = 232.615724 (without initial connection time)
```
tps показал наибольшее значение из всех ранее приведенных, однако не сказала бы, что он увеличился сильно. 

