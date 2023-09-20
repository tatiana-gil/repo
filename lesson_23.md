# Урок 23. Хранимые функции и процедуры.

__Создана БД trigger_db, схема pract_functions и установлен search_path на эту схему:__
```
postgres@ubuntu:~$ psql
postgres=# create database trigger_db;
CREATE DATABASE

postgres=# \c trigger_db 
You are now connected to database "trigger_db" as user "postgres".

trigger_db=# CREATE SCHEMA pract_functions;
CREATE SCHEMA

trigger_db=# SET search_path = pract_functions, publ;
SET
```
__В БД создана структура, описывающая товары (таблица goods) и продажи (таблица sales):__
```
trigger_db=# CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
CREATE TABLE

trigger_db=# INSERT INTO goods (goods_id, good_name, good_price)
VALUES  (1, 'Спички хозайственные', .50),
                (2, 'Автомобиль Ferrari FXX K', 185000000.01);
INSERT 0 2

trigger_db=# CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);
CREATE TABLE

trigger_db=# INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
INSERT 0 4
```
__Есть запрос для генерации отчета – сумма продаж по каждому товару:__
```
trigger_db=# SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

        good_name         |     sum      
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)
```
__БД была денормализована, создана таблица (витрина), структура которой повторяет структуру отчета:__
```
trigger_db=# CREATE TABLE good_sum_mart
(
        good_name   varchar(63) NOT NULL,
        sum_sale        numeric(16, 2)NOT NULL
);
CREATE TABLE
```
__Необходимо создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину).
Для этого сначала создана триггерная функция на INSERT:__
```
CREATE OR REPLACE FUNCTION on_new_sale() RETURNS TRIGGER AS
$BODY$
DECLARE
    update_good_name varchar(63);
  update_good_price numeric(12, 2);
  current_sum numeric(12, 2);
BEGIN

   -- Определяем название товара
   select good_name
      into update_good_name
   from goods where goods_id = new.good_id;
      
   -- Определяем цену товара
   select good_price
      into update_good_price
   from goods where goods_id = new.good_id;
   
   -- Если в отчете уже есть товар
   IF exists (select * from good_sum_mart where good_name = update_good_name) THEN
   
    -- Определяем текущую сумму продаж
  select sum_sale
       into current_sum
       from good_sum_mart 
    where good_name = update_good_name;
  
    -- Обновляем текущую сумму
    UPDATE good_sum_mart 
    SET sum_sale = current_sum + update_good_price * new.sales_qty 
    WHERE good_name = update_good_name;
   ELSE
       -- Или создаем норую запись в отчет
      INSERT INTO
        good_sum_mart(good_name, sum_sale)
        VALUES(update_good_name, update_good_price * new.sales_qty);
   END IF;
 RETURN new;
END;
$BODY$
language plpgsql;
```
__И создан триггер на инсерт:__ 
```
CREATE TRIGGER sales_new
BEFORE INSERT
ON sales
FOR EACH ROW EXECUTE PROCEDURE on_new_sale();
```

__Затем создана триггерная функция на DELETE:__
```
CREATE OR REPLACE FUNCTION on_remove_sale() RETURNS TRIGGER AS
$BODY$
DECLARE
    update_good_name varchar(63);
  update_good_price numeric(12, 2);
  current_sum numeric(12, 2);
BEGIN

   -- Определяем название товара
   select good_name
      into update_good_name
   from goods where goods_id = old.good_id;
      
   -- Определяем цену товара
   select good_price
      into update_good_price
   from goods where goods_id = old.good_id;
   
    -- Определяем текущую сумму продаж
   select sum_sale
   into current_sum
   from good_sum_mart 
   where good_name = update_good_name;
   
  -- Изменяем сумму продаж на новую
   UPDATE good_sum_mart 
   SET sum_sale = current_sum - update_good_price * old.sales_qty 
   WHERE good_name = update_good_name;
   RETURN old;
END;
$BODY$
language plpgsql;

CREATE FUNCTION
```
__И создан триггер на BEFORE DELETE:__
```
CREATE TRIGGER sales_delete
BEFORE DELETE
ON sales
FOR EACH ROW EXECUTE PROCEDURE on_remove_sale();
```

__И также создана триггерная функция на UPDATE:__
```
CREATE OR REPLACE FUNCTION on_update_sale() RETURNS TRIGGER AS
$BODY$
DECLARE
    update_good_name varchar(63);
    update_good_price numeric(12, 2);
    current_sum numeric(12, 2);
BEGIN

   -- Определяем название товара
   select good_name
    into update_good_name
   from pract_functions.goods where goods_id = old.good_id;

   -- Определяем цену товара
   select good_price
    into update_good_price
   from pract_functions.goods where goods_id = old.good_id;
     
   IF (OLD.good_id != NEW.good_id) THEN
      RAISE EXCEPTION 'Запрещено изменение good_id';
   END IF;
   
   IF (OLD.sales_id != NEW.sales_id) THEN
      RAISE EXCEPTION 'Запрещено изменение sales_id';
   END IF;
   
   -- Если изменилось количество
   IF NEW.sales_qty IS NOT NULL THEN
        -- Определяем текущую сумму продаж
     select sum_sale
     into current_sum
     from pract_functions.good_sum_mart 
     where good_name = update_good_name;
     
     -- Изменяем сумму продаж на новую
     UPDATE pract_functions.good_sum_mart
     SET sum_sale = current_sum + update_good_price * (new.sales_qty - old.sales_qty)
     WHERE good_name = update_good_name;
     RETURN new;
     
   END IF;

 RETURN new;
END;
$BODY$
language plpgsql;

CREATE FUNCTION
```
__И создан триггер на BEFORE UPDATE:__
```
CREATE TRIGGER sales_update
BEFORE UPDATE
ON sales
FOR EACH ROW EXECUTE PROCEDURE on_update_sale();

trigger_db=# select * from sales;
 sales_id | good_id |          sales_time           | sales_qty 
----------+---------+-------------------------------+-----------
        1 |       1 | 2023-09-18 16:12:23.319857+03 |        10
        2 |       1 | 2023-09-18 16:12:23.319857+03 |         1
        3 |       1 | 2023-09-18 16:12:23.319857+03 |       120
        4 |       2 | 2023-09-18 16:12:23.319857+03 |         1
(4 rows)
```

__Внесли данные инсертом для проверки того, что триггер работает:__
```
trigger_db=# INSERT INTO sales (good_id, sales_qty) VALUES (1, 50);
```
__При проверке таблицы good_sum_mart данные внеслись, но они не являются актуальными:__
```
trigger_db=# trigger_db=# select * from  good_sum_mart ;
      good_name       | sum_sale 
----------------------+----------
 Спички хозайственные |    25.00
(1 row)
```
__Поэтому внесем вручную данные, которые были в таблицах до того , как создали триггер:__
```
trigger_db=# INSERT INTO good_sum_mart (good_name, sum_sale)
VALUES             ('Автомобиль Ferrari FXX K', 185000000.01);

trigger_db=# UPDATE good_sum_mart SET sum_sale = 90.50
WHERE (good_name='Спички хозайственные');

UPDATE 1
```
__Теперь данные актуальны:__
```
trigger_db=# select * from good_sum_mart ;
        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        90.50
(2 rows)
```
__Проверим работу триггера на DELETE:__
```
trigger_db=# DELETE from sales where sales_id=5;

DELETE 1

trigger_db=# select * from  good_sum_mart ;

        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)

trigger_db=# SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G                             
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

        good_name         |     sum      
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)
```
__И проверим работу триггера на UPDATE:__
```
trigger_db=# UPDATE sales SET sales_qty = 90
WHERE (good_id=1);

trigger_db=# SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G                             
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

        good_name         |     sum      
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |       135.00
(2 rows)

trigger_db=# select * from  good_sum_mart ;
        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |       135.00
(2 rows)

trigger_db=# select * from sales;
 sales_id | good_id |          sales_time           | sales_qty 
----------+---------+-------------------------------+-----------
        4 |       2 | 2023-09-18 16:12:23.319857+03 |         1
        1 |       1 | 2023-09-18 16:12:23.319857+03 |        90
        2 |       1 | 2023-09-18 16:12:23.319857+03 |        90
        3 |       1 | 2023-09-18 16:12:23.319857+03 |        90
(4 rows)
```

__Задание со звездочкой   
Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?   
Подсказка: В реальной жизни возможны изменения цен.__

Схема витрина+триггер удобна тем, что в случае изменения цены не будет пересчета всей суммы, т.е. предыдущие транзакции останутся неизменными, сумма всех продаж будет оставаться той же, а товары по новой цене будут просто плюсоваться к сумме. В случае если бы был отчет , который пересобирался бы по требованию, то был бы пересчет всей суммы, что изменило бы реальную сумму продаж. Возможно в таком случае пришлось бы смотреть таблицу продаж на каждый момент времени, чтобы посчитать сумму, что значительно бы усложнило задачу. 



