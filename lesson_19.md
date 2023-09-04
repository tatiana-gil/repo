# Урок 19. Работа с индексами, join'ами, статистикой

## 1 вариант:
__Создать индексы на БД, которые ускорят доступ к данным.__

_Подготовка._

Создали новый кластер postgres 15 версии:
```
postgres@ubuntu:~$ pg_createcluster 15 index
postgres@ubuntu:~$ pg_ctlcluster 15 index start
postgres@ubuntu:~$ pg_lsclusters

Ver Cluster Port Status Owner    Data directory                Log file
...
15  index   5432 online postgres /var/lib/postgresql/15/index  /var/log/postgresql/postgresql-15-index.log
```

Создали БД и таблицу:
```
postgres=# create database index;
CREATE DATABASE

postgres=# \c index 
You are now connected to database "index" as user "postgres".
```
Создадим таблицу orders:
```
index=# create table orders (
    id int,
    user_id int,
    order_date date,
    status text,
    some_text text);
CREATE TABLE
```
Вставим данные в таблицу orders, столбец some_text будет содержать рандомные слова:
```
index=# insert into orders(id, user_id, order_date, status, some_text)
select generate_series, (random() * 70), date'2019-01-01' + (random() * 300)::int as order_date
        , (array['returned', 'completed', 'placed', 'shipped'])[(random() * 4)::int]
        , concat_ws(' ', (array['go', 'space', 'sun', 'London'])[(random() * 5)::int]
        , (array['the', 'capital', 'of', 'Great', 'Britain'])[(random() * 6)::int]
        , (array['some', 'another', 'example', 'with', 'words'])[(random() * 6)::int])
from generate_series(100001, 1000000);
INSERT 0 900000
```
Таблица состоит из данных следующего типа: 
```
index=# select * from orders limit 10;
   id   | user_id | order_date |  status   |       some_text        
--------+---------+------------+-----------+------------------------
 100001 |      17 | 2019-01-19 |           | space Britain another
 100002 |      68 | 2019-06-22 | returned  | go Great some
 100003 |      15 | 2019-10-05 | completed | London
 100004 |      41 | 2019-02-09 | completed | go
 100005 |      21 | 2019-10-11 | shipped   | sun capital
 100006 |      50 | 2019-05-25 | placed    | London Britain another
 100007 |      56 | 2019-03-04 |           | space with
 100008 |      43 | 2019-06-08 | placed    | sun capital
 100009 |      30 | 2019-04-28 | completed | space Britain some
 100010 |      16 | 2019-04-11 | placed    | space Great another
(10 rows)
```
__Создадим индекс на столбец id:__
```
index=# create index idx_ord_id on orders(id);
CREATE INDEX
```
Размер таблицы orders: 
```
index=# select pg_size_pretty(pg_table_size('orders'));
 pg_size_pretty 
----------------
 56 MB
(1 row)
```
Размер индекса idx_ord_id:
``` 
index=# select pg_size_pretty(pg_table_size('idx_ord_id'));
 pg_size_pretty 
----------------
 19 MB
(1 row)
```
Посмотрим explainы на селекты  к таблице orders с разными условиями: 
```
index=# explain
select *
from orders
where id < 500000;
                                    QUERY PLAN                                     
-----------------------------------------------------------------------------------
 Index Scan using idx_ord_id on orders  (cost=0.42..14564.68 rows=398929 width=34)
   Index Cond: (id < 500000)
(2 rows)

index=# explain
select *
from orders
order by id;
                                    QUERY PLAN                                     
-----------------------------------------------------------------------------------
 Index Scan using idx_ord_id on orders  (cost=0.42..30597.42 rows=900000 width=34)
(1 row)

index=# explain
select *
from orders
order by id desc;
                                         QUERY PLAN                                         
--------------------------------------------------------------------------------------------
 Index Scan Backward using idx_ord_id on orders  (cost=0.42..30597.42 rows=900000 width=34)
(1 row)

index=# explain
select * from orders where some_text ilike 'a%';
                                 QUERY PLAN                                  
-----------------------------------------------------------------------------
 Gather  (cost=1000.00..13788.60 rows=8911 width=34)
   Workers Planned: 2
   ->  Parallel Seq Scan on orders  (cost=0.00..11897.50 rows=3713 width=34)
         Filter: (some_text ~~* 'a%'::text)
(4 rows)
```
__Реализовать индекс на часть таблицы или индекс на поле с функцией__
```
index=# create index idx_order_some_text on orders(order_date);
CREATE INDEX
```
Посмотрим разные выводы селектов 
```
index=# select some_text, to_tsvector(some_text)
from orders;

index=# select some_text, to_tsvector(some_text) @@ to_tsquery('britains')
from orders;

index=# select some_text, to_tsvector(some_text) @@ to_tsquery('london & capital')
from orders;

index=# select some_text, to_tsvector(some_text) @@ to_tsquery('london | capital')
from orders;
```
Добавим доп. столбец some_text_lexeme с tsvector и заполним данными:
```
index=# alter table orders add column some_text_lexeme tsvector;
ALTER TABLE

index=# update orders
set some_text_lexeme = to_tsvector(some_text);
UPDATE 900000
```
Выполним explain к селекту с условием на столбце some_text_lexeme: 
```
index=# explain
select some_text
from orders
where some_text_lexeme @@ to_tsquery('britains');

                                   QUERY PLAN                                    
---------------------------------------------------------------------------------
 Gather  (cost=1000.00..131984.50 rows=149190 width=14)
   Workers Planned: 2
   ->  Parallel Seq Scan on orders  (cost=0.00..116065.50 rows=62162 width=14)
         Filter: (some_text_lexeme @@ to_tsquery('britains'::text))
 JIT:
   Functions: 4
   Options: Inlining false, Optimization false, Expressions true, Deforming true
(7 rows)
```
__Создадим GIN-индекс и выполним explain к нему:__
```
index=# CREATE INDEX search_index_ord ON orders USING GIN (some_text_lexeme);
CREATE INDEX

index=# explain
select *
from orders
where some_text_lexeme @@ to_tsquery('britains');

                                         QUERY PLAN                                          
---------------------------------------------------------------------------------------------
 Gather  (cost=2380.47..51245.13 rows=149190 width=62)
   Workers Planned: 2
   ->  Parallel Bitmap Heap Scan on orders  (cost=1380.47..35326.13 rows=62162 width=62)
         Recheck Cond: (some_text_lexeme @@ to_tsquery('britains'::text))
         ->  Bitmap Index Scan on search_index_ord  (cost=0.00..1343.18 rows=149190 width=0)
               Index Cond: (some_text_lexeme @@ to_tsquery('britains'::text))
(6 rows)
```
__Создадим составной индекс и выполним к нему explain:__
```
index=# create index idx_ord_order_date_status on orders(order_date, status);

CREATE INDEX
index=# explain
select order_date, status
from orders
where order_date between date'2020-01-01' and date'2020-02-01';

                                          QUERY PLAN                                          
----------------------------------------------------------------------------------------------
 Index Only Scan using idx_ord_order_date_status on orders  (cost=0.42..4.44 rows=1 width=12)
   Index Cond: ((order_date >= '2020-01-01'::date) AND (order_date <= '2020-02-01'::date))
(2 rows)
```
cost показывает стоимость запуска и общую стоимость каждого узла плана, а также рассчитанное число строк и ширину каждой строки. 
Наибольшую стоимость видим при использовании GIN-индекса cost=2380.47..51245.13 rows=149190 width=62, т.к. полнотекстовый индекс довольно тяжелый
Для получения первой строки затратится 2380.47, для получения всех строк затратится 51245.13. rows - примерное кол-во возвращаемых строк при выполнении операции = 149190. width - средний размер 1-й строки в байтах = 62. 

## 2 вариант:

Создадим новую бд join_practice и несколько таблиц в ней:
```
index=# create  database join_practice;
CREATE DATABASE

index=# \c join_practice 
You are now connected to database "join_practice" as user "postgres".

join_practice=# create table bus (id serial,route text,id_model int,id_driver int); 
CREATE TABLE

join_practice=# create table model_bus (id serial,name text);
CREATE TABLE

join_practice=# create table driver (id serial,first_name text,second_name text);
CREATE TABLE
```
Добавим данных в таблицы:
```
join_practice=# insert into bus values (1,'Москва-Болшево',1,1),(2,'Москва-Пушкино',1,2),(3,'Москва-Ярославль',2,3),(4,'Москва-Кострома',2,4),(5,'Москва-Волгорад',3,5),(6,'Москва-Иваново',null,null);
INSERT 0 6

join_practice=# insert into model_bus values(1,'ПАЗ'),(2,'ЛИАЗ'),(3,'MAN'),(4,'МАЗ'),(5,'НЕФАЗ');
INSERT 0 5

join_practice=# insert into driver values(1,'Иван','Иванов'),(2,'Петр','Петров'),(3,'Савелий','Сидоров'),(4,'Антон','Шторкин'),(5,'Олег','Зажигаев'),(6,'Аркадий','Паровозов');
INSERT 0 6
```
Данные в таблицах выглядят след. образом: 
```
join_practice=# select * from bus;

 id |      route       | id_model | id_driver 
----+------------------+----------+-----------
  1 | Москва-Болшево   |        1 |         1
  2 | Москва-Пушкино   |        1 |         2
  3 | Москва-Ярославль |        2 |         3
  4 | Москва-Кострома  |        2 |         4
  5 | Москва-Волгорад  |        3 |         5
  6 | Москва-Иваново   |          |          
(6 rows)

join_practice=# select * from model_bus;

 id | name  
----+-------
  1 | ПАЗ
  2 | ЛИАЗ
  3 | MAN
  4 | МАЗ
  5 | НЕФАЗ
(5 rows)

join_practice=# select * from driver;

 id | first_name | second_name 
----+------------+-------------
  1 | Иван       | Иванов
  2 | Петр       | Петров
  3 | Савелий    | Сидоров
  4 | Антон      | Шторкин
  5 | Олег       | Зажигаев
  6 | Аркадий    | Паровозов
(6 rows)
```
__Реализовать прямое соединение двух или более таблиц__
```
join_practice=# select *
from bus b
join model_bus mb
    on b.id_model=mb.id;
	
 id |      route       | id_model | id_driver | id | name 
----+------------------+----------+-----------+----+------
  1 | Москва-Болшево   |        1 |         1 |  1 | ПАЗ
  2 | Москва-Пушкино   |        1 |         2 |  1 | ПАЗ
  3 | Москва-Ярославль |        2 |         3 |  2 | ЛИАЗ
  4 | Москва-Кострома  |        2 |         4 |  2 | ЛИАЗ
  5 | Москва-Волгорад  |        3 |         5 |  3 | MAN
(5 rows)
```
__Реализовать левостороннее (или правостороннее) соединение двух или более таблиц__
```
join_practice=# select *
from bus b
left join model_bus mb
    on b.id_model=mb.id;
	
 id |      route       | id_model | id_driver | id | name 
----+------------------+----------+-----------+----+------
  1 | Москва-Болшево   |        1 |         1 |  1 | ПАЗ
  2 | Москва-Пушкино   |        1 |         2 |  1 | ПАЗ
  3 | Москва-Ярославль |        2 |         3 |  2 | ЛИАЗ
  4 | Москва-Кострома  |        2 |         4 |  2 | ЛИАЗ
  5 | Москва-Волгорад  |        3 |         5 |  3 | MAN
  6 | Москва-Иваново   |          |           |    | 
(6 rows)
```
__Реализовать кросс соединение двух или более таблиц__
```
join_practice=# select *
from bus b
cross join model_bus mb;

 id |      route       | id_model | id_driver | id | name  
----+------------------+----------+-----------+----+-------
  1 | Москва-Болшево   |        1 |         1 |  1 | ПАЗ
  2 | Москва-Пушкино   |        1 |         2 |  1 | ПАЗ
  3 | Москва-Ярославль |        2 |         3 |  1 | ПАЗ
  4 | Москва-Кострома  |        2 |         4 |  1 | ПАЗ
  5 | Москва-Волгорад  |        3 |         5 |  1 | ПАЗ
  6 | Москва-Иваново   |          |           |  1 | ПАЗ
  1 | Москва-Болшево   |        1 |         1 |  2 | ЛИАЗ
  2 | Москва-Пушкино   |        1 |         2 |  2 | ЛИАЗ
  3 | Москва-Ярославль |        2 |         3 |  2 | ЛИАЗ
  4 | Москва-Кострома  |        2 |         4 |  2 | ЛИАЗ
  5 | Москва-Волгорад  |        3 |         5 |  2 | ЛИАЗ
  6 | Москва-Иваново   |          |           |  2 | ЛИАЗ
  1 | Москва-Болшево   |        1 |         1 |  3 | MAN
  2 | Москва-Пушкино   |        1 |         2 |  3 | MAN
  3 | Москва-Ярославль |        2 |         3 |  3 | MAN
  4 | Москва-Кострома  |        2 |         4 |  3 | MAN
  5 | Москва-Волгорад  |        3 |         5 |  3 | MAN
  6 | Москва-Иваново   |          |           |  3 | MAN
  1 | Москва-Болшево   |        1 |         1 |  4 | МАЗ
  2 | Москва-Пушкино   |        1 |         2 |  4 | МАЗ
  3 | Москва-Ярославль |        2 |         3 |  4 | МАЗ
  4 | Москва-Кострома  |        2 |         4 |  4 | МАЗ
  5 | Москва-Волгорад  |        3 |         5 |  4 | МАЗ
  6 | Москва-Иваново   |          |           |  4 | МАЗ
  1 | Москва-Болшево   |        1 |         1 |  5 | НЕФАЗ
  2 | Москва-Пушкино   |        1 |         2 |  5 | НЕФАЗ
  3 | Москва-Ярославль |        2 |         3 |  5 | НЕФАЗ
  4 | Москва-Кострома  |        2 |         4 |  5 | НЕФАЗ
  5 | Москва-Волгорад  |        3 |         5 |  5 | НЕФАЗ
  6 | Москва-Иваново   |          |           |  5 | НЕФАЗ
(30 rows)
```
__Реализовать полное соединение двух или более таблиц__
```
join_practice=# select *
from bus b
full join model_bus mb on b.id_model=mb.id;

 id |      route       | id_model | id_driver | id | name  
----+------------------+----------+-----------+----+-------
  1 | Москва-Болшево   |        1 |         1 |  1 | ПАЗ
  2 | Москва-Пушкино   |        1 |         2 |  1 | ПАЗ
  3 | Москва-Ярославль |        2 |         3 |  2 | ЛИАЗ
  4 | Москва-Кострома  |        2 |         4 |  2 | ЛИАЗ
  5 | Москва-Волгорад  |        3 |         5 |  3 | MAN
  6 | Москва-Иваново   |          |           |    | 
    |                  |          |           |  4 | МАЗ
    |                  |          |           |  5 | НЕФАЗ
(8 rows)
```
__Реализовать запрос, в котором будут использованы разные типы соединений__
Также рассмотрим порядок join (параметры планировщика)
```
join_practice=# CREATE TABLE test AS
    SELECT    (random()*100)::int AS id,
             'product ' || id AS product
    FROM generate_series(1, 10000) AS id;

SELECT 10000
join_practice=# select * from test;

join_practice=# create table test_2 (id int);
CREATE TABLE

join_practice=# insert into test_2 values (1);
INSERT 0 1

join_practice=# SET enable_hashjoin = on;
SET

join_practice=# SET enable_mergejoin = on;
SET

join_practice=# SET enable_nestloop = on;
SET

join_practice=# set join_collapse_limit to 8;
SET

join_practice=# set join_collapse_limit to 1;
SET

join_practice=# explain
select *
from test_2 t2
inner join test t1
on t2.id=t1.id
inner join test_2 t3
on t3.id=t2.id;

                                   QUERY PLAN                                   
--------------------------------------------------------------------------------
 Merge Join  (cost=1186.95..27815.33 rows=1625625 width=24)
   Merge Cond: (t2.id = t3.id)
   ->  Merge Join  (cost=1007.17..2932.42 rows=127500 width=20)
         Merge Cond: (t2.id = t1.id)
         ->  Sort  (cost=179.78..186.16 rows=2550 width=4)
               Sort Key: t2.id
               ->  Seq Scan on test_2 t2  (cost=0.00..35.50 rows=2550 width=4)
         ->  Sort  (cost=827.39..852.39 rows=10000 width=16)
               Sort Key: t1.id
               ->  Seq Scan on test t1  (cost=0.00..163.00 rows=10000 width=16)
   ->  Sort  (cost=179.78..186.16 rows=2550 width=4)
         Sort Key: t3.id
         ->  Seq Scan on test_2 t3  (cost=0.00..35.50 rows=2550 width=4)
(13 rows)
```

