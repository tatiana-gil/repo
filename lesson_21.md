# Урок 21. Секционирование

## Задача: Секционировать большую таблицу из демо базы flights


Скачали файл с БД demo, распаковали и создали БД в соответствии с содержимым файла:
```
postgres@ubuntu:~$ wget https://edu.postgrespro.com/demo-big-en.zip

postgres@ubuntu:~$ sudo apt-get install unzip

postgres@ubuntu:~$ psql -f ./datasets/demo-big-en-20170815.sql
...
```
```
postgres=# \l
                                                   List of databases
     Name      |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges   
---------------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 demo          | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | 
...

postgres=# \c demo 
You are now connected to database "demo" as user "postgres".
```
Имеются таблицы bookings и tickets, которые будут использоваться:
```
demo=# \d bookings
                        Table "bookings.bookings"
    Column    |           Type           | Collation | Nullable | Default 
--------------+--------------------------+-----------+----------+---------
 book_ref     | character(6)             |           | not null | 
 book_date    | timestamp with time zone |           | not null | 
 total_amount | numeric(10,2)            |           | not null | 
Indexes:
    "bookings_pkey" PRIMARY KEY, btree (book_ref)
Referenced by:
    TABLE "tickets" CONSTRAINT "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)

demo=# \d tickets
                        Table "bookings.tickets"
     Column     |         Type          | Collation | Nullable | Default 
----------------+-----------------------+-----------+----------+---------
 ticket_no      | character(13)         |           | not null | 
 book_ref       | character(6)          |           | not null | 
 passenger_id   | character varying(20) |           | not null | 
 passenger_name | text                  |           | not null | 
 contact_data   | jsonb                 |           |          | 
Indexes:
    "tickets_pkey" PRIMARY KEY, btree (ticket_no)
Foreign-key constraints:
    "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)
Referenced by:
    TABLE "ticket_flights" CONSTRAINT "ticket_flights_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES tickets(ticket_no)
```
Делаем секционирование на таблице bookings по диапазону дат:
```
demo=# CREATE TABLE bookings_range (
       book_ref     character(6),
       book_date    timestamptz,
       total_amount numeric(10,2)
   ) PARTITION BY RANGE(book_date);
CREATE TABLE

demo=# CREATE TABLE bookings_range_201706 PARTITION OF bookings_range 
       FOR VALUES FROM ('2017-06-01'::timestamptz) TO ('2017-07-01'::timestamptz);
CREATE TABLE

demo=# CREATE TABLE bookings_range_201707 PARTITION OF bookings_range 
       FOR VALUES FROM ('2017-07-01'::timestamptz) TO ('2017-08-01'::timestamptz);
CREATE TABLE
demo=# CREATE TABLE bookings_range_201606 PARTITION OF bookings_range 
       FOR VALUES FROM ('2016-06-01'::timestamptz) TO ('2016-07-01'::timestamptz);
CREATE TABLE

demo=# CREATE TABLE bookings_range_201607 PARTITION OF bookings_range 
       FOR VALUES FROM ('2016-07-01'::timestamptz) TO ('2016-08-01'::timestamptz);
CREATE TABLE

demo=# CREATE TABLE bookings_range_201608 PARTITION OF bookings_range 
       FOR VALUES FROM ('2016-08-01'::timestamptz) TO ('2016-09-01'::timestamptz);
CREATE TABLE

demo=# CREATE TABLE bookings_range_201609 PARTITION OF bookings_range 
       FOR VALUES FROM ('2016-09-01'::timestamptz) TO ('2016-10-01'::timestamptz);
CREATE TABLE

demo=# CREATE TABLE bookings_range_201610 PARTITION OF bookings_range 
       FOR VALUES FROM ('2016-10-01'::timestamptz) TO ('2016-11-01'::timestamptz);
CREATE TABLE
```
Также можно использовать выражения, чтобы указывать границы секции: 
```
demo=# CREATE TABLE bookings_range_201708 PARTITION OF bookings_range 
       FOR VALUES FROM (to_timestamp('01.08.2017','DD.MM.YYYY')) 
                    TO (to_timestamp('01.09.2017','DD.MM.YYYY'));
CREATE TABLE
```
Посмотрим на таблицу bookings_range:
```
demo=# \d+ bookings_range
                                          Partitioned table "bookings.bookings_range"
    Column    |           Type           | Collation | Nullable | Default | Storage  | Compression | Stats target | Description 
--------------+--------------------------+-----------+----------+---------+----------+-------------+--------------+-------------
 book_ref     | character(6)             |           |          |         | extended |             |              | 
 book_date    | timestamp with time zone |           |          |         | plain    |             |              | 
 total_amount | numeric(10,2)            |           |          |         | main     |             |              | 
Partition key: RANGE (book_date)
Partitions: bookings_range_201606 FOR VALUES FROM ('2016-06-01 00:00:00+03') TO ('2016-07-01 00:00:00+03'),
            bookings_range_201607 FOR VALUES FROM ('2016-07-01 00:00:00+03') TO ('2016-08-01 00:00:00+03'),
            bookings_range_201608 FOR VALUES FROM ('2016-08-01 00:00:00+03') TO ('2016-09-01 00:00:00+03'),
            bookings_range_201609 FOR VALUES FROM ('2016-09-01 00:00:00+03') TO ('2016-10-01 00:00:00+03'),
            bookings_range_201610 FOR VALUES FROM ('2016-10-01 00:00:00+03') TO ('2016-11-01 00:00:00+03'),
            bookings_range_201706 FOR VALUES FROM ('2017-06-01 00:00:00+03') TO ('2017-07-01 00:00:00+03'),
            bookings_range_201707 FOR VALUES FROM ('2017-07-01 00:00:00+03') TO ('2017-08-01 00:00:00+03'),
            bookings_range_201708 FOR VALUES FROM ('2017-08-01 00:00:00+03') TO ('2017-09-01 00:00:00+03')

demo=# SET constraint_exclusion = OFF;
SET
```
При попытке заполнения с автоматической раскладкой по секциям получали ошибку:
```
demo=# INSERT INTO bookings_range SELECT * FROM bookings;
ERROR:  no partition of relation "bookings_range" found for row
DETAIL:  Partition key of the failing row contains (book_date) = (2017-01-08 19:45:00+03).
```
Необходимо было добавить еще некоторое кол-во диапозонов дат, которых не хватало: 
```
demo=# CREATE TABLE bookings_range_201701 PARTITION OF bookings_range 
       FOR VALUES FROM ('2017-01-01'::timestamptz) TO ('2017-02-01'::timestamptz);
CREATE TABLE

demo=# INSERT INTO bookings_range SELECT * FROM bookings;
ERROR:  no partition of relation "bookings_range" found for row
DETAIL:  Partition key of the failing row contains (book_date) = (2017-05-20 18:45:00+03).

demo=# CREATE TABLE bookings_range_201705 PARTITION OF bookings_range 
       FOR VALUES FROM ('2017-05-01'::timestamptz) TO ('2017-06-01'::timestamptz);
CREATE TABLE

demo=# INSERT INTO bookings_range SELECT * FROM bookings;
ERROR:  no partition of relation "bookings_range" found for row
DETAIL:  Partition key of the failing row contains (book_date) = (2016-12-12 15:02:00+03).

demo=# CREATE TABLE bookings_range_201612 PARTITION OF bookings_range 
       FOR VALUES FROM ('2016-12-01'::timestamptz) TO ('2017-01-01'::timestamptz);
CREATE TABLE

demo=# INSERT INTO bookings_range SELECT * FROM bookings;
ERROR:  no partition of relation "bookings_range" found for row
DETAIL:  Partition key of the failing row contains (book_date) = (2017-03-08 22:18:00+03).

demo=# CREATE TABLE bookings_range_201703 PARTITION OF bookings_range 
       FOR VALUES FROM ('2017-03-01'::timestamptz) TO ('2017-04-01'::timestamptz);
CREATE TABLE

CREATE TABLE bookings_range_201702 PARTITION OF bookings_range 
       FOR VALUES FROM ('2017-02-01'::timestamptz) TO ('2017-03-01'::timestamptz);

CREATE TABLE bookings_range_201704 PARTITION OF bookings_range 
       FOR VALUES FROM ('2017-04-01'::timestamptz) TO ('2017-05-01'::timestamptz);

CREATE TABLE bookings_range_201709 PARTITION OF bookings_range 
       FOR VALUES FROM ('2017-09-01'::timestamptz) TO ('2017-10-01'::timestamptz);
	   
CREATE TABLE bookings_range_201710 PARTITION OF bookings_range 
       FOR VALUES FROM ('2017-10-01'::timestamptz) TO ('2017-11-01'::timestamptz);
	   
CREATE TABLE bookings_range_201711 PARTITION OF bookings_range 
       FOR VALUES FROM ('2017-11-01'::timestamptz) TO ('2017-12-01'::timestamptz);
	  
CREATE TABLE bookings_range_201601 PARTITION OF bookings_range 
       FOR VALUES FROM ('2016-01-01'::timestamptz) TO ('2016-02-01'::timestamptz);
	   
CREATE TABLE bookings_range_201602 PARTITION OF bookings_range 
       FOR VALUES FROM ('2016-02-01'::timestamptz) TO ('2016-03-01'::timestamptz);
	   
CREATE TABLE bookings_range_201603 PARTITION OF bookings_range 
       FOR VALUES FROM ('2016-03-01'::timestamptz) TO ('2016-04-01'::timestamptz);

CREATE TABLE bookings_range_201604 PARTITION OF bookings_range 
       FOR VALUES FROM ('2016-04-01'::timestamptz) TO ('2016-05-01'::timestamptz);

CREATE TABLE bookings_range_201605 PARTITION OF bookings_range 
       FOR VALUES FROM ('2016-05-01'::timestamptz) TO ('2016-06-01'::timestamptz);

CREATE TABLE bookings_range_201611 PARTITION OF bookings_range 
       FOR VALUES FROM ('2016-11-01'::timestamptz) TO ('2016-12-01'::timestamptz);
```
После чего удалось выполнить заполнение с автоматической раскладкой по секциям:
```
demo=# INSERT INTO bookings_range SELECT * FROM bookings;
INSERT 0 2111110
```
За декларативным синтаксисом по-прежнему скрываются наследуемые таблицы, поэтому распределение строк по секциям можно посмотреть запросом:
```
demo=# SELECT tableoid::regclass, count(*) FROM bookings_range GROUP BY tableoid;
       tableoid        | count  
-----------------------+--------
 bookings_range_201706 | 165150
 bookings_range_201707 | 171760
 bookings_range_201607 |  10878
 bookings_range_201608 | 168329
 bookings_range_201609 | 165421
 bookings_range_201610 | 170874
 bookings_range_201708 |  88423
 bookings_range_201701 | 171183
 bookings_range_201705 | 171009
 bookings_range_201612 | 171265
 bookings_range_201703 | 171332
 bookings_range_201702 | 154579
 bookings_range_201704 | 165438
 bookings_range_201611 | 165469
(14 rows)
```
В родительской таблице данных нет:
```
demo=# SELECT * FROM ONLY bookings_range;
 book_ref | book_date | total_amount 
----------+-----------+--------------
(0 rows)
```
Посмотрим план запроса с условием: 
```
demo=# EXPLAIN (COSTS OFF) 
   SELECT * FROM bookings_range WHERE book_date = '2016-07-01'::timestamptz;
                                 QUERY PLAN                                 
----------------------------------------------------------------------------
 Seq Scan on bookings_range_201607 bookings_range
   Filter: (book_date = '2016-07-01 00:00:00+03'::timestamp with time zone)
(2 rows)
```
В следующем плане запроса вместо константы используется функция to_timestamp с категорией изменчивости STABLE:
```
demo=# EXPLAIN (COSTS OFF) 
   SELECT * FROM bookings_range WHERE book_date = to_timestamp('01.07.2016','DD.MM.YYYY');
                                        QUERY PLAN                                        
------------------------------------------------------------------------------------------
 Gather
   Workers Planned: 2
   ->  Parallel Append
         Subplans Removed: 22
         ->  Parallel Seq Scan on bookings_range_201607 bookings_range_1
               Filter: (book_date = to_timestamp('01.07.2016'::text, 'DD.MM.YYYY'::text))
(6 rows)
```
Значение функции вычисляется при инициализации плана запроса, а из Subplans Removed часть секций исключается из просмотра.
Это работает только для SELECTа. При изменении данных исключение секций на основе значений STABLE функций пока не реализовано:
```
demo=# EXPLAIN (COSTS OFF) 
   DELETE FROM bookings_range WHERE book_date = to_timestamp('01.07.2016','DD.MM.YYYY');
                                        QUERY PLAN                                        
------------------------------------------------------------------------------------------
 Delete on bookings_range
   Delete on bookings_range_201601 bookings_range
   Delete on bookings_range_201602 bookings_range
   Delete on bookings_range_201603 bookings_range
   Delete on bookings_range_201604 bookings_range
   Delete on bookings_range_201605 bookings_range
   Delete on bookings_range_201606 bookings_range
   Delete on bookings_range_201607 bookings_range_1
   Delete on bookings_range_201608 bookings_range
   Delete on bookings_range_201609 bookings_range
   Delete on bookings_range_201610 bookings_range
   Delete on bookings_range_201611 bookings_range
   Delete on bookings_range_201612 bookings_range
   Delete on bookings_range_201701 bookings_range
   Delete on bookings_range_201702 bookings_range
   Delete on bookings_range_201703 bookings_range
   Delete on bookings_range_201704 bookings_range
   Delete on bookings_range_201705 bookings_range
   Delete on bookings_range_201706 bookings_range
   Delete on bookings_range_201707 bookings_range
   Delete on bookings_range_201708 bookings_range
   Delete on bookings_range_201709 bookings_range
   Delete on bookings_range_201710 bookings_range
   Delete on bookings_range_201711 bookings_range
   ->  Append
         Subplans Removed: 22
         ->  Seq Scan on bookings_range_201607 bookings_range_1
               Filter: (book_date = to_timestamp('01.07.2016'::text, 'DD.MM.YYYY'::text))
(28 rows)
```
Поэтому используем константы:
```
demo=# EXPLAIN (COSTS OFF) 
   DELETE FROM bookings_range WHERE book_date = '2016-07-01'::timestamptz;
                                    QUERY PLAN                                    
----------------------------------------------------------------------------------
 Delete on bookings_range
   Delete on bookings_range_201607 bookings_range_1
   ->  Seq Scan on bookings_range_201607 bookings_range_1
         Filter: (book_date = '2016-07-01 00:00:00+03'::timestamp with time zone)
(4 rows)
```
Выполним следующий запрос с сортировкой. В плане запроса появляется SORT и высокая начальная стоимость плана в 135806.38:
```
demo=# EXPLAIN SELECT * FROM bookings_range ORDER BY book_date;
                                                        QUERY PLAN                                                         
---------------------------------------------------------------------------------------------------------------------------
 Gather Merge  (cost=136806.41..342959.83 rows=1766906 width=21)
   Workers Planned: 2
   ->  Sort  (cost=135806.38..138015.02 rows=883453 width=21)
         Sort Key: bookings_range.book_date
         ->  Parallel Append  (cost=0.00..30433.56 rows=883453 width=21)
               ->  Parallel Seq Scan on bookings_range_201707 bookings_range_19  (cost=0.00..2105.35 rows=101035 width=21)
               ->  Parallel Seq Scan on bookings_range_201703 bookings_range_15  (cost=0.00..2099.84 rows=100784 width=21)
               ->  Parallel Seq Scan on bookings_range_201612 bookings_range_12  (cost=0.00..2098.44 rows=100744 width=21)
               ->  Parallel Seq Scan on bookings_range_201701 bookings_range_13  (cost=0.00..2097.96 rows=100696 width=21)
               ->  Parallel Seq Scan on bookings_range_201705 bookings_range_17  (cost=0.00..2095.94 rows=100594 width=21)
               ->  Parallel Seq Scan on bookings_range_201610 bookings_range_10  (cost=0.00..2094.14 rows=100514 width=21)
               ->  Parallel Seq Scan on bookings_range_201608 bookings_range_8  (cost=0.00..2063.17 rows=99017 width=21)
               ->  Parallel Seq Scan on bookings_range_201611 bookings_range_11  (cost=0.00..2027.35 rows=97335 width=21)
               ->  Parallel Seq Scan on bookings_range_201704 bookings_range_16  (cost=0.00..2027.16 rows=97316 width=21)
               ->  Parallel Seq Scan on bookings_range_201609 bookings_range_9  (cost=0.00..2027.06 rows=97306 width=21)
               ->  Parallel Seq Scan on bookings_range_201706 bookings_range_18  (cost=0.00..2023.47 rows=97147 width=21)
               ->  Parallel Seq Scan on bookings_range_201702 bookings_range_14  (cost=0.00..1894.29 rows=90929 width=21)
               ->  Parallel Seq Scan on bookings_range_201708 bookings_range_20  (cost=0.00..1084.14 rows=52014 width=21)
               ->  Parallel Seq Scan on bookings_range_201607 bookings_range_7  (cost=0.00..133.99 rows=6399 width=21)
               ->  Parallel Seq Scan on bookings_range_201601 bookings_range_1  (cost=0.00..16.00 rows=600 width=52)
               ->  Parallel Seq Scan on bookings_range_201602 bookings_range_2  (cost=0.00..16.00 rows=600 width=52)
               ->  Parallel Seq Scan on bookings_range_201603 bookings_range_3  (cost=0.00..16.00 rows=600 width=52)
               ->  Parallel Seq Scan on bookings_range_201604 bookings_range_4  (cost=0.00..16.00 rows=600 width=52)
               ->  Parallel Seq Scan on bookings_range_201605 bookings_range_5  (cost=0.00..16.00 rows=600 width=52)
               ->  Parallel Seq Scan on bookings_range_201606 bookings_range_6  (cost=0.00..16.00 rows=600 width=52)
               ->  Parallel Seq Scan on bookings_range_201709 bookings_range_21  (cost=0.00..16.00 rows=600 width=52)
               ->  Parallel Seq Scan on bookings_range_201710 bookings_range_22  (cost=0.00..16.00 rows=600 width=52)
               ->  Parallel Seq Scan on bookings_range_201711 bookings_range_23  (cost=0.00..16.00 rows=600 width=52)
(28 rows)
```
Создадим индекс по book_date:
```
demo=# CREATE INDEX book_date_idx ON bookings_range(book_date);
CREATE INDEX
```
Вместо одного глобального индекса, создаются индексы в каждой секции:
```
demo=# \di bookings_range*
                                     List of relations
  Schema  |                Name                 | Type  |  Owner   |         Table         
----------+-------------------------------------+-------+----------+-----------------------
 bookings | bookings_range_201601_book_date_idx | index | postgres | bookings_range_201601
 bookings | bookings_range_201602_book_date_idx | index | postgres | bookings_range_201602
 bookings | bookings_range_201603_book_date_idx | index | postgres | bookings_range_201603
 bookings | bookings_range_201604_book_date_idx | index | postgres | bookings_range_201604
 bookings | bookings_range_201605_book_date_idx | index | postgres | bookings_range_201605
 bookings | bookings_range_201606_book_date_idx | index | postgres | bookings_range_201606
 bookings | bookings_range_201607_book_date_idx | index | postgres | bookings_range_201607
 bookings | bookings_range_201608_book_date_idx | index | postgres | bookings_range_201608
 bookings | bookings_range_201609_book_date_idx | index | postgres | bookings_range_201609
 bookings | bookings_range_201610_book_date_idx | index | postgres | bookings_range_201610
 bookings | bookings_range_201611_book_date_idx | index | postgres | bookings_range_201611
 bookings | bookings_range_201612_book_date_idx | index | postgres | bookings_range_201612
 bookings | bookings_range_201701_book_date_idx | index | postgres | bookings_range_201701
 bookings | bookings_range_201702_book_date_idx | index | postgres | bookings_range_201702
 bookings | bookings_range_201703_book_date_idx | index | postgres | bookings_range_201703
 bookings | bookings_range_201704_book_date_idx | index | postgres | bookings_range_201704
 bookings | bookings_range_201705_book_date_idx | index | postgres | bookings_range_201705
 bookings | bookings_range_201706_book_date_idx | index | postgres | bookings_range_201706
 bookings | bookings_range_201707_book_date_idx | index | postgres | bookings_range_201707
 bookings | bookings_range_201708_book_date_idx | index | postgres | bookings_range_201708
 bookings | bookings_range_201709_book_date_idx | index | postgres | bookings_range_201709
 bookings | bookings_range_201710_book_date_idx | index | postgres | bookings_range_201710
 bookings | bookings_range_201711_book_date_idx | index | postgres | bookings_range_201711
(23 rows)
```
Предыдущий запрос с сортировкой теперь может использовать индекс по ключу секционирования и выдавать результат из разных секций сразу в отсортированном виде. 
Узел SORT не нужен и для выдачи первой строки результата требуются минимальные затраты (cost в 5.47):
```
demo=# EXPLAIN SELECT * FROM bookings_range ORDER BY book_date;
                                                                    QUERY PLAN                                                                    
--------------------------------------------------------------------------------------------------------------------------------------------------
 Append  (cost=5.47..110277.58 rows=2120290 width=21)
   ->  Index Scan using bookings_range_201601_book_date_idx on bookings_range_201601 bookings_range_1  (cost=0.15..59.45 rows=1020 width=52)
   ->  Index Scan using bookings_range_201602_book_date_idx on bookings_range_201602 bookings_range_2  (cost=0.15..59.45 rows=1020 width=52)
   ->  Index Scan using bookings_range_201603_book_date_idx on bookings_range_201603 bookings_range_3  (cost=0.15..59.45 rows=1020 width=52)
   ->  Index Scan using bookings_range_201604_book_date_idx on bookings_range_201604 bookings_range_4  (cost=0.15..59.45 rows=1020 width=52)
   ->  Index Scan using bookings_range_201605_book_date_idx on bookings_range_201605 bookings_range_5  (cost=0.15..59.45 rows=1020 width=52)
   ->  Index Scan using bookings_range_201606_book_date_idx on bookings_range_201606 bookings_range_6  (cost=0.15..59.45 rows=1020 width=52)
   ->  Index Scan using bookings_range_201607_book_date_idx on bookings_range_201607 bookings_range_7  (cost=0.29..547.43 rows=10878 width=21)
   ->  Index Scan using bookings_range_201608_book_date_idx on bookings_range_201608 bookings_range_8  (cost=0.29..7909.03 rows=168329 width=21)
   ->  Index Scan using bookings_range_201609_book_date_idx on bookings_range_201609 bookings_range_9  (cost=0.29..7765.44 rows=165421 width=21)
   ->  Index Scan using bookings_range_201610_book_date_idx on bookings_range_201610 bookings_range_10  (cost=0.29..8023.32 rows=170874 width=21)
   ->  Index Scan using bookings_range_201611_book_date_idx on bookings_range_201611 bookings_range_11  (cost=0.29..7769.99 rows=165469 width=21)
   ->  Index Scan using bookings_range_201612_book_date_idx on bookings_range_201612 bookings_range_12  (cost=0.29..8037.25 rows=171265 width=21)
   ->  Index Scan using bookings_range_201701_book_date_idx on bookings_range_201701 bookings_range_13  (cost=0.29..8036.01 rows=171183 width=21)
   ->  Index Scan using bookings_range_201702_book_date_idx on bookings_range_201702 bookings_range_14  (cost=0.29..7258.94 rows=154579 width=21)
   ->  Index Scan using bookings_range_201703_book_date_idx on bookings_range_201703 bookings_range_15  (cost=0.29..8042.00 rows=171332 width=21)
   ->  Index Scan using bookings_range_201704_book_date_idx on bookings_range_201704 bookings_range_16  (cost=0.29..7765.76 rows=165438 width=21)
   ->  Index Scan using bookings_range_201705_book_date_idx on bookings_range_201705 bookings_range_17  (cost=0.29..8029.21 rows=171009 width=21)
   ->  Index Scan using bookings_range_201706_book_date_idx on bookings_range_201706 bookings_range_18  (cost=0.29..7753.54 rows=165150 width=21)
   ->  Index Scan using bookings_range_201707_book_date_idx on bookings_range_201707 bookings_range_19  (cost=0.29..8064.52 rows=171760 width=21)
   ->  Index Scan using bookings_range_201708_book_date_idx on bookings_range_201708 bookings_range_20  (cost=0.29..4138.64 rows=88423 width=21)
   ->  Index Scan using bookings_range_201709_book_date_idx on bookings_range_201709 bookings_range_21  (cost=0.15..59.45 rows=1020 width=52)
   ->  Index Scan using bookings_range_201710_book_date_idx on bookings_range_201710 bookings_range_22  (cost=0.15..59.45 rows=1020 width=52)
   ->  Index Scan using bookings_range_201711_book_date_idx on bookings_range_201711 bookings_range_23  (cost=0.15..59.45 rows=1020 width=52)
(24 rows)
```
Созданные таким образом индексы на секциях поддерживаются централизованно. При добавлении новой секции на ней автоматически будет создан индекс. 

Удалить индекс только одной секции нельзя:
```
demo=# DROP INDEX bookings_range_201706_book_date_idx;
ERROR:  cannot drop index bookings_range_201706_book_date_idx because index book_date_idx requires it
HINT:  You can drop index book_date_idx instead.
```
Удаление индекса возможно только полностью:
```
demo=# DROP INDEX book_date_idx;
DROP INDEX
```
При создании индекса на секционированной таблице нельзя указать CONCURRENTLY.
Но можно поступить следующим образом. Сначала создаем индекс только на основной таблице, он получит статус invalid:
```
demo=# CREATE INDEX book_date_idx ON ONLY bookings_range(book_date);
CREATE INDEX

demo=# SELECT indisvalid FROM pg_index WHERE indexrelid::regclass::text = 'book_date_idx';
 indisvalid 
------------
 f
(1 row)
```
Затем создаем индексы на всех секциях с опцией CONCURRENTLY:
```
CREATE INDEX CONCURRENTLY book_date_201601_idx ON bookings_range_201601 (book_date);
CREATE INDEX CONCURRENTLY book_date_201602_idx ON bookings_range_201602 (book_date);
CREATE INDEX CONCURRENTLY book_date_201603_idx ON bookings_range_201603 (book_date);
CREATE INDEX CONCURRENTLY book_date_201604_idx ON bookings_range_201604 (book_date);
CREATE INDEX CONCURRENTLY book_date_201605_idx ON bookings_range_201605 (book_date);
CREATE INDEX CONCURRENTLY book_date_201606_idx ON bookings_range_201606 (book_date);
CREATE INDEX CONCURRENTLY book_date_201607_idx ON bookings_range_201607 (book_date);
CREATE INDEX CONCURRENTLY book_date_201608_idx ON bookings_range_201608 (book_date);
CREATE INDEX CONCURRENTLY book_date_201609_idx ON bookings_range_201609 (book_date);
CREATE INDEX CONCURRENTLY book_date_201610_idx ON bookings_range_201610 (book_date);
CREATE INDEX CONCURRENTLY book_date_201611_idx ON bookings_range_201611 (book_date);
CREATE INDEX CONCURRENTLY book_date_201612_idx ON bookings_range_201612 (book_date);

CREATE INDEX CONCURRENTLY book_date_201701_idx ON bookings_range_201701 (book_date);
CREATE INDEX CONCURRENTLY book_date_201702_idx ON bookings_range_201702 (book_date);
CREATE INDEX CONCURRENTLY book_date_201703_idx ON bookings_range_201703 (book_date);
CREATE INDEX CONCURRENTLY book_date_201704_idx ON bookings_range_201704 (book_date);
CREATE INDEX CONCURRENTLY book_date_201705_idx ON bookings_range_201705 (book_date);
CREATE INDEX CONCURRENTLY book_date_201706_idx ON bookings_range_201706 (book_date);
CREATE INDEX CONCURRENTLY book_date_201707_idx ON bookings_range_201707 (book_date);
CREATE INDEX CONCURRENTLY book_date_201708_idx ON bookings_range_201708 (book_date);
CREATE INDEX CONCURRENTLY book_date_201709_idx ON bookings_range_201709 (book_date);
CREATE INDEX CONCURRENTLY book_date_201710_idx ON bookings_range_201710 (book_date);
CREATE INDEX CONCURRENTLY book_date_201711_idx ON bookings_range_201711 (book_date);
```
Теперь подключаем локальные индексы к глобальному:
```
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201701_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201702_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201703_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201704_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201705_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201706_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201707_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201708_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201709_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201710_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201711_idx;

ALTER INDEX book_date_idx ATTACH PARTITION book_date_201601_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201602_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201603_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201604_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201605_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201606_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201607_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201608_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201609_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201610_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201611_idx;
ALTER INDEX book_date_idx ATTACH PARTITION book_date_201612_idx;
```
Как только все индексные секции будут подключены, основной индекс изменит свой статус:
```
demo=# SELECT indisvalid FROM pg_index WHERE indexrelid::regclass::text = 'book_date_idx';
 indisvalid 
------------
 t
(1 row)
```
Подключение и отключение секций

Автоматическое создание секций не предусмотрено. Поэтому их нужно создавать заранее, до того как в таблицу начнут добавляться записи с новыми значениями ключа секционирования.

Будем создавать новую секцию во время работы других транзакций с таблицей, заодно посмотрим на блокировки:
```
demo=# BEGIN;
demo=*# SELECT count(*) FROM bookings_range
    WHERE book_date = to_timestamp('01.08.2016','DD.MM.YYYY');
 count 
-------
     3
(1 row)

demo=*# SELECT relation::regclass::text, mode FROM pg_locks 
    WHERE pid = pg_backend_pid() AND relation::regclass::text LIKE 'bookings%';
       relation        |      mode       
-----------------------+-----------------
 bookings_range_201608 | AccessShareLock
 bookings_range_201607 | AccessShareLock
 bookings_range_201606 | AccessShareLock
 bookings_range_201605 | AccessShareLock
 bookings_range_201604 | AccessShareLock
 bookings_range_201603 | AccessShareLock
 bookings_range_201602 | AccessShareLock
 bookings_range_201601 | AccessShareLock
 bookings_range        | AccessShareLock
 bookings_range_201709 | AccessShareLock
 bookings_range_201609 | AccessShareLock
 bookings_range_201711 | AccessShareLock
 bookings_range_201702 | AccessShareLock
 bookings_range_201701 | AccessShareLock
 bookings_range_201708 | AccessShareLock
 bookings_range_201611 | AccessShareLock
 bookings_range_201612 | AccessShareLock
 bookings_range_201704 | AccessShareLock
 bookings_range_201705 | AccessShareLock
 bookings_range_201703 | AccessShareLock
 bookings_range_201610 | AccessShareLock
 bookings_range_201706 | AccessShareLock
 bookings_range_201710 | AccessShareLock
 bookings_range_201707 | AccessShareLock
(24 rows)
```
Блокировка AccessShareLock накладывается на основную таблицу, все секции и индексы в начале выполнения оператора. 
Вычисление функции to_timestamp и исключение секций происходит позже.
Если бы вместо функции использовалась константа, то блокировалась бы только основная таблица и секция bookings_range_201707.
Поэтому при возможности указывать в запросе константы — это следует делать, иначе количество строк в pg_locks будет увеличиваться пропорционально количеству секций, что может привести к необходимости увеличения max_locks_per_transaction.

