# Урок 14.  Виды и устройство репликации в PostgreSQL. Практика применения
* ## Подготовка 
__Развернули 3 ВМ с сетевыми параметрами NAT и internal network. Выполнили предварительные настройки серверов для удобной работы.__

IP ВМ 1: 192.168.56.102
IP ВМ 2: 192.168.56.103
IP ВМ 3: 192.168.56.104

__На каждой ВМ установили postgres 15:__
```
postgres@ubuntu:~$ 
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
```
__И развернули по кластеру:__

_На 1 ВМ:_
```
postgres@ubuntu:~$ pg_createcluster 15 1_for_repl
postgres@ubuntu:~$ pg_ctlcluster 15 1_for_repl start

postgres@ubuntu:~$ pg_lsclusters 
Ver Cluster    Port Status Owner    Data directory                    Log file
14  main       5432 online postgres /var/lib/postgresql/14/main       /var/log/postgresql/postgresql-14-main.log
15  1_for_repl 5434 online postgres /var/lib/postgresql/15/1_for_repl /var/log/postgresql/postgresql-15-1_for_repl.log
15  main       5433 online postgres /var/lib/postgresql/15/main       /var/log/postgresql/postgresql-15-main.log
```
_На 2 ВМ:_
```
postgres@ubuntu:~$ pg_createcluster 15 2_for_repl
postgres@ubuntu:~$ pg_ctlcluster 15 2_for_repl start

postgres@ubuntu:~$ pg_lsclusters 

Ver Cluster    Port Status Owner    Data directory                    Log file
14  main       5432 down   postgres /var/lib/postgresql/14/main       /var/log/postgresql/postgresql-14-main.log
15  2_for_repl 5434 online postgres /var/lib/postgresql/15/2_for_repl /var/log/postgresql/postgresql-15-2_for_repl.log
15  main       5433 online postgres /var/lib/postgresql/15/main       /var/log/postgresql/postgresql-15-main.log
```
_На 3 ВМ:_
```
postgres@ubuntu:~$ pg_createcluster 15 3_for_repl
postgres@ubuntu:~$ pg_ctlcluster 15 3_for_repl start

postgres@ubuntu:~$ pg_lsclusters 
Ver Cluster    Port Status Owner    Data directory                    Log file
14  main       5432 online postgres /var/lib/postgresql/14/main       /var/log/postgresql/postgresql-14-main.log
15  3_for_repl 5434 online postgres /var/lib/postgresql/15/3_for_repl /var/log/postgresql/postgresql-15-3_for_repl.log
15  main       5433 online postgres /var/lib/postgresql/15/main       /var/log/postgresql/postgresql-15-main.log
```

__На мастере изменили wal_level на logical:__
```
postgres@ubuntu:~$ psql -p 5434
postgres=# show wal_level;

 wal_level 
-----------
 replica

postgres=# ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM
```
__В postgresql.conf раскомментировали и поменяли параметр listen_addresses на '*':__

postgres@ubuntu:~$ vi /etc/postgresql/15/1_for_repl/postgresql.conf
listen_addresses = '*'

В pg_hba.conf добавили строки:

postgres@ubuntu:~$ vi /etc/postgresql/15/1_for_repl/pg_hba.conf 

#for otus logical replication
host    all             all             0.0.0.0/0           scram-sha-256
host    all             all             ::/0                scram-sha-256

__Сделали рестарт кластера для применения новых параметров:__
```
postgres@ubuntu:~$ pg_ctlcluster 15 1_for_repl restart 
```
__Также обновили postgresql.conf и pg_hba.conf для кластеров постгреса на 2-й и 3-й ВМ.__


* ## На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение.
__Создали новую бд log_replica, в ней таблицы:__
```
postgres=# create database log_replica;
postgres=# \c log_replica 

log_replica=# create table test as 
select 
  generate_series(1,10) as id,
  md5(random()::text)::char(10) as random_symbol;
SELECT 10

log_replica=# create table test2 (id serial, random_symbol char(10));  
CREATE TABLE

log_replica=# select * from test;
 id | random_symbol 
----+---------------
  1 | 134cc05aa7
  2 | e07c4f8223
  3 | 148d97940d
  4 | 9b24ad75d6
  5 | be24bb911e
  6 | faacaa1c3e
  7 | 33175a099a
  8 | bddde2c10c
  9 | a845359ea0
 10 | f223746675
(10 rows)

log_replica=# select * from test2 ;
 id | random_symbol 
----+---------------
(0 rows)
```

* ## Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.
__Создали публикацию:__
```
log_replica=# CREATE PUBLICATION pub_test FOR TABLE test;
CREATE PUBLICATION
```
__Просмотр публикации:__
```
log_replica=# \dRp+
                            Publication pub_test
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root 
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test"
```
__Задали пароль__
```
\password 
```
__Создали подписку на 2-й вм (как выяснилось позднее, необходимо также создать бд и таблицу):__
```
postgres=# create database log_replica;
CREATE DATABASE

postgres=# \c log_replica 

log_replica=# create table test (id serial, random_symbol char(10));  
CREATE TABLE

log_replica=# CREATE SUBSCRIPTION test_sub 
CONNECTION 'host=192.168.56.102 port=5434 user=postgres password=2121More35 dbname=log_replica' 
PUBLICATION pub_test WITH (copy_data = true);
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION

log_replica=# \dRs
            List of subscriptions
   Name   |  Owner   | Enabled | Publication 
----------+----------+---------+-------------
 test_sub | postgres | t       | {pub_test}
(1 row)

log_replica=# SELECT * FROM pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16399
subname               | test_sub
pid                   | 54570
relid                 | 
received_lsn          | 0/19B0590
last_msg_send_time    | 2023-08-03 16:25:54.684361+03
last_msg_receipt_time | 2023-08-03 16:25:54.685598+03
latest_end_lsn        | 0/19B0590
latest_end_time       | 2023-08-03 16:25:54.684361+03
```
* ## На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение.

__Создали таблицу test2:__
```
log_replica=# create table test2 as 
select 
  generate_series(1,10) as id,
  md5(random()::text)::char(10) as random_symbol;
SELECT 10
```
_Таблица test уже создана в предыдущем шаге_

* ## Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.

__На 2 ВМ создали публикацию таблицы test2:__
```
log_replica=# CREATE PUBLICATION pub_test2 FOR TABLE test2;
CREATE PUBLICATION
```
__На 1 ВМ подписались на ранее созданную публикацию:
```
log_replica=# CREATE SUBSCRIPTION test2_sub 
CONNECTION 'host=192.168.56.103 port=5434 user=postgres password=2121More35 dbname=log_replica' 
PUBLICATION pub_test2 WITH (copy_data = true);
NOTICE:  created replication slot "test2_sub" on publisher
CREATE SUBSCRIPTION

log_replica=# \dRs
            List of subscriptions
   Name    |  Owner   | Enabled | Publication 
-----------+----------+---------+-------------
 test2_sub | postgres | t       | {pub_test2}
(1 row)

log_replica=# SELECT * FROM pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16411
subname               | test2_sub
pid                   | 61663
relid                 | 
received_lsn          | 0/198DC20
last_msg_send_time    | 2023-08-03 17:02:52.41991+03
last_msg_receipt_time | 2023-08-03 17:02:52.420075+03
latest_end_lsn        | 0/198DC20
latest_end_time       | 2023-08-03 17:02:52.41991+03
```
* ## 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2).

__На 3 ВМ установили wal_level на logical:__
```
postgres=# show wal_level;

 wal_level 
-----------
 replica

postgres=# ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM

postgres@ubuntu:~$ pg_ctlcluster 15 3_for_repl restart
```
__Затем создали бд log_replica, создали таблицы test и test2 для чтения и сделали подписки на публикации из вм1 и вм2:__
```
postgres=# create database log_replica;
CREATE DATABASE

postgres=# \c log_replica 

log_replica=# create table test (id serial, random_symbol char(10));
CREATE TABLE

log_replica=# create table test2 (id serial, random_symbol char(10));
CREATE TABLE

log_replica=# CREATE SUBSCRIPTION test_sub 
CONNECTION 'host=192.168.56.102 port=5434 user=postgres password=2121More35 dbname=log_replica' 
PUBLICATION pub_test WITH (copy_data = true);
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION

log_replica=# CREATE SUBSCRIPTION test2_sub 
CONNECTION 'host=192.168.56.103 port=5434 user=postgres password=2121More35 dbname=log_replica' 
PUBLICATION pub_test2 WITH (copy_data = true);
NOTICE:  created replication slot "test2_sub" on publisher
CREATE SUBSCRIPTION
```




