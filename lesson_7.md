# Логический уровень PostgreSQL

__1 создайте новый кластер PostgresSQL 15__
```
root@ubuntu:~# sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14
```
__2 зайдите в созданный кластер под пользователем postgres__
```
postgres@ubuntu:~$ psql
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.
```
__3 создайте новую базу данных testdb__
```
postgres=# create database testdb;
CREATE DATABASE
```
__4 зайдите в созданную базу данных под пользователем postgres__
```
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
```
__5 создайте новую схему testnm__
```
testdb=# create schema testnm;
CREATE SCHEMA
```
__6 создайте новую таблицу t1 с одной колонкой c1 типа integer__
```
testdb=# create table t1 (c1 integer);
CREATE TABLE
```
__7 вставьте строку со значением c1=1__
```
testdb=# insert into t1(c1) values (1);
INSERT 0 1
```
__8 создайте новую роль readonly__
```
testdb=# create role readonly with password 'Pass123';
CREATE ROLE
```
__9 дайте новой роли право на подключение к базе данных testdb__
```
testdb=# grant connect on database testdb to readonly;
GRANT
```
__10 дайте новой роли право на использование схемы testnm__
```
testdb=# grant usage on schema testnm to readonly;
GRANT
```
__11 дайте новой роли право на select для всех таблиц схемы testnm__
```
testdb=# grant select on all tables in schema testnm to readonly;
GRANT
```
__12 создайте пользователя testread с паролем test123__
```
testdb=# create user testread with password 'test123';
CREATE ROLE
```
__13 дайте роль readonly пользователю testread__
```
testdb=# grant readonly to testread;
GRANT ROLE
```
__14 зайдите под пользователем testread в базу данных testdb__
```
testdb=# \c testdb testread
connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "testread"
Previous connection kept

postgres=# show hba_file;
              hba_file               
-------------------------------------
 /etc/postgresql/15/main/pg_hba.conf
(1 row)

postgres@ubuntu:/etc$ vi /etc/postgresql/15/main/pg_hba.conf
```
добавлена строка:
```
local   all             all                                     scram-sha-256
```
и закомментирована строка:
```
#local   all             all                                     peer
```
перечитан  файл конфигурации + проверка на ошибки:
```
postgres@ubuntu:/etc$ psql -c "select pg_reload_conf()"
 pg_reload_conf 
----------------
 t
(1 row)

postgres@ubuntu:/etc$ psql -c "SELECT line_number, error FROM pg_hba_file_rules WHERE error IS NOT NULL"
 line_number | error 
-------------+-------
(0 rows)

postgres=# \c testdb testread
Password for user testread: 
You are now connected to database "testdb" as user "testread".
```
__15 сделайте select * from t1;__
```
testdb=> select * from t1;
ERROR:  permission denied for table t1
```
__16 получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)__  
нет  
__17 напишите что именно произошло в тексте домашнего задания__  
ошибка о недостатке прав на таблицу  
__18 у вас есть идеи почему? ведь права то дали?__  
мы раздавали права на схему testnm, но таблица, к которой пытаемся обратиться, находится в схеме public  
__19 посмотрите на список таблиц__
```
testdb=> \dt

        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
```
__20 подсказка в шпаргалке под пунктом 20__
__21 а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)__
```
testdb=> show search_path;
   search_path   
-----------------
 "$user", public
(1 row)
```
элемент $user ссылается на схему с именем текущего пользователя. Нов нашем случае такой схемы не существует, ссылка на неё игнорируется. Поэтому идет отсылка на схему public.  
__22 вернитесь в базу данных testdb под пользователем postgres__
```
testdb=> \c testdb postgres 
You are now connected to database "testdb" as user "postgres".
```
__23 удалите таблицу t1__
```
testdb=# drop table t1;
DROP TABLE
```
__24 создайте ее заново но уже с явным указанием имени схемы testnm__
```
testdb=# create table testnm.t1 (c1 integer);
CREATE TABLE
```
__25 вставьте строку со значением c1=1__
```
testdb=# insert into testnm.t1(c1) values (1);
INSERT 0 1
```
__26 зайдите под пользователем testread в базу данных testdb__
```
testdb=# \c testdb testread
Password for user testread: 
You are now connected to database "testdb" as user "testread".
```
__27 сделайте select * from testnm.t1;__
```
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
__28 получилось?__  
нет  
__29 есть идеи почему?__  
потому что пересоздали таблицу после того как были розданы права ON ALL TABLES.  
__30 как сделать так чтобы такое больше не повторялось?__
```
testdb=> \c testdb postgres
You are now connected to database "testdb" as user "postgres".
testdb=# ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly; 
ALTER DEFAULT PRIVILEGES
```
__31 сделайте select * from testnm.t1;__
```
testdb=# \c testdb testread;
Password for user testread: 
You are now connected to database "testdb" as user "testread".
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
__32 получилось?__  
нет  
__33 есть идеи почему? если нет - смотрите шпаргалку__  
потому что ALTER default будет действовать для новых таблиц а grant SELECT on all TABLEs in SCHEMA testnm TO readonly отработал только для существующих на тот момент времени. надо сделать снова или grant SELECT или пересоздать таблицу
```
testdb=> \c testdb postgres 
You are now connected to database "testdb" as user "postgres".
testdb=# grant select on all tables in schema testnm to readonly;
GRANT
```
__31 сделайте select * from testnm.t1;__
```
testdb=# \c testdb testread;
Password for user testread: 
You are now connected to database "testdb" as user "testread".
testdb=> select * from testnm.t1;
 c1 
----
  1
(1 row)
```
__32 получилось?__  
да  
__33 ура!__  
ура-а-а!  
__34 теперь попробуйте выполнить команду create table t2(c1 integer); 
insert into t2 values (2);__
```
testdb=> create table t2(c1 integer);
ERROR:  permission denied for schema public
LINE 1: create table t2(c1 integer);
                     ^
testdb=> insert into t2 values (2);
ERROR:  relation "t2" does not exist
LINE 1: insert into t2 values (2);
                    ^
```
__35 а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?__  
да, все верно  
__36 есть идеи как убрать эти права?__  
прав нет, т.к используется 15 версия постгреса, а в ней права на CREATE TABLE по умолчанию отозваны у схемы PUBLIC, только USAGE.  
__37 если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды__  
если бы использовалась более ранняя версия постгреса, то права можно было бы отозвать так:
```
\c testdb postgres; 
REVOKE CREATE on SCHEMA public FROM public; 
REVOKE ALL on DATABASE testdb FROM public; 
\c testdb testread; 
```
__38 теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);__
```
testdb=> create table t3(c1 integer);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
```
__39 расскажите что получилось и почему__  
прав по прежнему нет, все отлично
