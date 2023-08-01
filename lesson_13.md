# Урок 13. Резервное копирование и восстановление 

__Создали новый кластер postgres 15 версии:__
```
postgres@ubuntu:/var/lib/postgresql/15/summ$ pg_createcluster 15 backup
postgres@ubuntu:/var/lib/postgresql/15/summ$ pg_ctlcluster 15 backup start

postgres@ubuntu:/var/lib/postgresql/15/summ$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory                Log file
15  backup  5435 online postgres /var/lib/postgresql/15/backup /var/log/postgresql/postgresql-15-backup.log
```
__Создали БД, схему и в ней таблицу:__
```
postgres@ubuntu:/var/lib/postgresql/15/summ$ psql -p 5435
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))

postgres=# create database bkp;
CREATE DATABASE

postgres=# \c bkp 
You are now connected to database "bkp" as user "postgres".

bkp=# create schema schema_bkp;
CREATE SCHEMA

bkp=# create table schema_bkp.table_bkp as 
select 
  generate_series(1,100) as id,
  md5(random()::text)::char(10) as random_symbol;
SELECT 100
```
__Заполнили таблицу автосгенерированными 100 записями__
```
bkp=# select * from schema_bkp.table_bkp limit 10;
 id | random_symbol 
----+---------------
  1 | 25c098fc4e
  2 | 2a0ddd86bb
  3 | 83fa66eda3
  4 | ecd67cacb9
  5 | 86c5b05ecf
  6 | f05f06ce01
  7 | bd78d9d4b4
  8 | b468e17293
  9 | 2e1df74fca
 10 | 8dacd38f97
(10 rows)

bkp=# select count(*) from schema_bkp.table_bkp;
 count 
-------
   100
(1 row)
```
__Под линукс пользователем Postgres создали каталог для бэкапов:__
```
postgres@ubuntu:/tmp$ mkdir backup
drwxrwxr-x 2 postgres postgres 4096 авг  1 20:36 backup
```
__Сделали логический бэкап таблицы используя утилиту COPY__
```
bkp=# \copy schema_bkp.table_bkp to '/tmp/backup/tb_bkp.sql';
COPY 100

postgres@ubuntu:/tmp/backup$ ls -l
total 4
-rw-rw-r-- 1 postgres postgres 1392 авг  1 20:40 tb_bkp.sql

postgres@ubuntu:/tmp/backup$ cat tb_bkp.sql 
1	25c098fc4e
2	2a0ddd86bb
3	83fa66eda3
4	ecd67cacb9
5	86c5b05ecf
...
```
__Восстановили во вторую созданную таблицу данные из бэкапа:__
```
bkp=# create table schema_bkp.table_bkp_2 (id serial, random_symbol char(10)); 

bkp=# \copy schema_bkp.table_bkp_2 from '/tmp/backup/tb_bkp.sql';
COPY 100

bkp=# select * from schema_bkp.table_bkp_2 limit 10;
 id | random_symbol 
----+---------------
  1 | 25c098fc4e
  2 | 2a0ddd86bb
  3 | 83fa66eda3
  4 | ecd67cacb9
  5 | 86c5b05ecf
  6 | f05f06ce01
  7 | bd78d9d4b4
  8 | b468e17293
  9 | 2e1df74fca
 10 | 8dacd38f97
(10 rows)
```
__Используя утилиту pg_dump создали бэкап в кастомном сжатом формате двух таблиц:__
```
postgres@ubuntu:/tmp/backup$ pg_dump -t 'schema_bkp.table_bkp*' -d bkp --create -Fc > /tmp/backup/arch.gz

postgres@ubuntu:/tmp/backup$ ls -l
-rw-rw-r-- 1 postgres postgres 4591 авг  1 21:13 arch.gz
```
__Используя утилиту pg_restore восстановили в новую БД только вторую таблицу:_
```
postgres=# create database bkp_2;
CREATE DATABASE
bkp_2=# create schema schema_bkp;
CREATE SCHEMA

pg_restore -t table_bkp_2 -d bkp_2 /tmp/backup/arch.gz
```
_Проверка:_
```
postgres@ubuntu:/tmp/backup$ psql
postgres=# \c bkp_2 
bkp_2=# \dt schema_bkp.*
              List of relations
   Schema   |    Name     | Type  |  Owner   
------------+-------------+-------+----------
 schema_bkp | table_bkp_2 | table | postgres
(1 row)

bkp_2=# select * from schema_bkp.table_bkp_2 limit 10;
 id | random_symbol 
----+---------------
  1 | 25c098fc4e
  2 | 2a0ddd86bb
  3 | 83fa66eda3
  4 | ecd67cacb9
  5 | 86c5b05ecf
  6 | f05f06ce01
  7 | bd78d9d4b4
  8 | b468e17293
  9 | 2e1df74fca
 10 | 8dacd38f97
(10 rows)
```
__Проблемы, с которыми столкнулась:__

* Изначально делала pg_dump с помощью gzip > /tmp/backup/2_tb.gz, однако рестор такой тип архива не признает: 
```
postgres@ubuntu:/tmp/backup$ pg_dump -t 'schema_bkp.table_bkp*' -d bkp --create | gzip > /tmp/backup/2_tb.gz

postgres@ubuntu:/tmp/backup$ ls -l
-rw-rw-r-- 1 postgres postgres 1752 авг  1 20:59 2_tb.gz

postgres@ubuntu:/tmp/backup$ pg_restore -t table_bkp_2 -d bkp_2 /tmp/backup/2_tb.gz
pg_restore: error: input file does not appear to be a valid archive
```
* В команде pg_restore при указании таблицы нельзя указать конкретную схему: 
```
postgres@ubuntu:/tmp/backup$ pg_restore -t schema_bkp.table_bkp_2 -d bkp_2 /tmp/backup/arch.gz
```
_Примечание из postgrespro:_
Этот флаг (-t) действует не совсем так, как флаг -t утилиты pg_dump. В настоящее время pg_restore не поддерживает выбор объектов по маске, а также не позволяет указать имя схемы с -t. 

* Когда восстанавливаешь таблицу необходимо, чтобы в новой бд была также создана схема, в которую будет идти рестор. Иначе получаем такую ошибку: 
```
postgres@ubuntu:/tmp/backup$ pg_restore -t table_bkp_2 -d bkp_2 /tmp/backup/arch.gz
pg_restore: error: could not execute query: ERROR:  schema "schema_bkp" does not exist
LINE 1: CREATE TABLE schema_bkp.table_bkp_2 (
                     ^
Command was: CREATE TABLE schema_bkp.table_bkp_2 (
    id integer NOT NULL,
    random_symbol character(10)
);
pg_restore: error: could not execute query: ERROR:  schema "schema_bkp" does not exist
Command was: ALTER TABLE schema_bkp.table_bkp_2 OWNER TO postgres;
pg_restore: error: could not execute query: ERROR:  schema "schema_bkp" does not exist
Command was: COPY schema_bkp.table_bkp_2 (id, random_symbol) FROM stdin;
pg_restore: warning: errors ignored on restore: 3
```
