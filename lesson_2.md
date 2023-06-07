# SQL и реляционные СУБД. Введение в PostgreSQL 

* Создан новый проект в Яндекс облаке
* Создан инстанс виртуальной машины с дефолтными параметрами
* Добавлен ssh ключ
* Подключено удаленным ssh (первая сессия)
* Подключено вторым ssh (вторая сессия)
* Запущен везде psql из под пользователя postgres\
___postgres@ubuntu:~$ psql___   
* Выключен auto commit   
___postgres=# \echo :AUTOCOMMIT___   
```		on 	```\
___postgres=# \set AUTOCOMMIT OFF___   
___postgres=# \echo :AUTOCOMMIT___   
```		OFF```
* В первой сессии создана новая таблица и заполнена данными\
___postgres=# create table persons(id serial, first_name text, second_name text);___\
___insert into persons(first_name, second_name) values('ivan', 'ivanov');___\
___insert into persons(first_name, second_name) values('petr', 'petrov');___\
___commit;___\
___postgres=# select * from persons;___\
```
		id | first_name | second_name
		---+------------+-------------
		 1 | ivan       | ivanov
		 2 | petr       | petrov
		(2 rows)
```
* Текущий уровень изоляции:\
___postgres=*# show transaction isolation level;___\
```
		transaction_isolation
		-----------------------
		read committed
		(1 row)
```
* Начаты новые транзакции в обоих сессиях с дефолтным (не меняя) уровнем изоляции
* В первой сессии добавлена новая запись:\
___insert into persons(first_name, second_name) values('sergey', 'sergeev');___   
* Во второй сессии:\
___postgres=# select * from persons;___\
```
		id | first_name | second_name_
		---+------------+-------------
		 1 | ivan       | ivanov
		 2 | petr       | petrov
		(2 rows)
```
* Видите ли вы новую запись и если да то почему?\
___Ответ:___ _Нет, т.к. на уровне изоляции read committed, при SELECTе видны только те данные, которые были зафиксированы до начала запроса. В нашем случае первая транзакция как раз не зафиксирована._\
* Завершена первая транзакция\
___postgres=*# commit;___
* Выполнен запрос во второй сессии:\
___postgres=*# select * from persons;___\
```
		id | first_name | second_name_
		 --+------------+-------------
		 1 | ivan       | ivanov
		 2 | petr       | petrov
		 3 | sergey     | sergeev
		(3 rows)
```
* Видите ли вы новую запись и если да то почему?\
___Ответ:___ _Новая запись видна второй сессией, т.к. первая теперь выполнила commit - завершила транзакцию._   
* Завершена транзакция во второй сессии\
___postgres=*# commit;___\
_----------------------------------------------------------------------_   
* Начаты новые транзакции с уровнем изоляции repeatable read:\
___postgres=# set transaction isolation level repeatable read;___\
___postgres=*# show transaction isolation level;___   
```
		transaction_isolation
		-----------------------
		repeatable read
```
* В первой сессии добавлена новая запись:\
___postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');___
* Выполнен запрос во второй сессии:\
___postgres=*# select * from persons;___\
```
		id | first_name | second_name
		---+------------+-------------
		 1 | ivan       | ivanov
		 2 | petr       | petrov
		 3 | sergey     | sergeev
		(3 rows)
```
* Видите ли вы новую запись и если да то почему?\
___Ответ:___ _Нет, в уровне изоляции repeatable read  видны только те данные, которые были зафиксированы до начала транзакции, но не видны незафиксированные данные и изменения, произведённые другими транзакциями в процессе выполнения данной транзакции. В данном случае этот уровень изоляции позволяет избегать фантомного чтения (когда при повторном чтении в рамках одной транзакции одна и та же выборка дает разные множества строк)._   
* Завершена первая транзакцию:\
___postgres=*# commit;___   
* Выполнен запрос во второй сессии:\
___postgres=*# select * from persons;___\
```
		 id | first_name | second_name
		----+------------+-------------
		  1 | ivan       | ivanov
		  2 | petr       | petrov
		  3 | sergey     | sergeev
		(3 rows)
```
* Видите ли вы новую запись и если да то почему?\
___Ответ:___ _нет, в уровне изоляции repeatable read не видны изменения, произведённые другими транзакциями в процессе выполнения данной транзакции._    
* Завершена вторая транзакция\
___postgres=*# commit;___   
* Выполнен запрос во второй сессии:\
___postgres=# select * from persons;___\
```
		 id | first_name | second_name
		----+------------+-------------
		  1 | ivan       | ivanov
		  2 | petr       | petrov
		  3 | sergey     | sergeev
		  4 | sveta      | svetova
		(4 rows)
```
* Видите ли вы новую запись и если да то почему?\
___Ответ:___ _Да, т.к. транзакция с уровнем изоляции  repeatable read была завершена. На данный момент уровень изоляции вернулся к стандартному  read committed._\
___postgres=*# show transaction isolation level;___\
```
		transaction_isolation
		-----------------------
		read committed
```