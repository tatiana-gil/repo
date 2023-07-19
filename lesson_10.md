# Урок 10. Блокировки
### Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

* Создадим новую бд locks и таблица accounts в ней:
```
postgres=# create database locks;
CREATE DATABASE

postgres=# \c locks 
You are now connected to database "locks" as user "postgres".

locks=# CREATE TABLE accounts(
  acc_no integer PRIMARY KEY,
  amount numeric
);
INSERT INTO accounts VALUES (1,1000.00), (2,2000.00), (3,3000.00);
CREATE TABLE
INSERT 0 3
```
* Включим параметр log_lock_waits и установим deadlock_timeout на 200ms:
```
locks=# ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM

locks=# ALTER SYSTEM SET deadlock_timeout ='200ms';
ALTER SYSTEM

locks=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

locks=# SHOW deadlock_timeout;
 deadlock_timeout 
------------------
 200ms
(1 row)
```
* Моделируем ситуацию с занесением блокировки в журнал:

Сессия 1:
```
locks=# BEGIN;
UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
BEGIN
UPDATE 1
```
Сессия 2: (зависает в ожидании)
```
locks=# BEGIN;
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
BEGIN
```
Сессия 1:
```
locks=*# commit;
COMMIT
```
Сессия 2:
```
...
UPDATE 1

locks=*# commit;
COMMIT
```
* Посмотрим логфайл, в котором появилась запись о блокировке:
```
postgres@ubuntu:/var/log/postgresql$ vi /var/log/postgresql/postgresql-15-main.log

...
2023-07-19 13:06:26.321 MSK [4350] postgres@locks LOG:  process 4350 still waiting for ShareLock on transaction 1508629 after 200.531 ms
2023-07-19 13:06:26.321 MSK [4350] postgres@locks DETAIL:  Process holding the lock: 4204. Wait queue: 4350.
2023-07-19 13:06:26.321 MSK [4350] postgres@locks CONTEXT:  while updating tuple (0,6) in relation "accounts"
2023-07-19 13:06:26.321 MSK [4350] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2023-07-19 13:06:32.786 MSK [4350] postgres@locks LOG:  process 4350 acquired ShareLock on transaction 1508629 after 6665.934 ms
2023-07-19 13:06:32.786 MSK [4350] postgres@locks CONTEXT:  while updating tuple (0,6) in relation "accounts"
2023-07-19 13:06:32.786 MSK [4350] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
```
### Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.
* Создадим новую таблицу accounts:
```
locks=# CREATE TABLE accounts(
  acc_no integer PRIMARY KEY,
  amount numeric
);
CREATE TABLE

locks=# INSERT INTO accounts VALUES (1, 100.00), (2, 200.00), (3, 300.00);
INSERT 0 3

locks=# select * from accounts;
 acc_no | amount 
--------+--------
      1 | 100.00
      2 | 200.00
      3 | 300.00
(3 rows)
```
* Заапдейтим одну и ту же строку тремя разными транзакциями:   
Сессия 1:
```
locks=# BEGIN;
locks=*# SELECT pg_backend_pid();
 pg_backend_pid 
----------------
           2608
(1 row)

locks=*# UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
UPDATE 1

locks=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 2608;
   locktype    |   relation    | virtxid |   xid   |       mode       | granted 
---------------+---------------+---------+---------+------------------+---------
 relation      | pg_locks      |         |         | AccessShareLock  | t
 relation      | accounts_pkey |         |         | RowExclusiveLock | t
 relation      | accounts      |         |         | RowExclusiveLock | t
 virtualxid    |               | 4/9     |         | ExclusiveLock    | t
 transactionid |               |         | 1508645 | ExclusiveLock    | t
(5 rows)
```
В первой сессии с pid = 2608 видим блокировки:
- AccessShareLock, который получается от команды SELECT на таблицу pg_locks, обычная блокировка при чтении. Будет конфливтовать только с режимом ACCESS EXCLUSIVE;
- RowExclusiveLock, блокировки возникли при апдейте строки в таблице accounts. Конфликтует с режимами блокировки SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE и ACCESS EXCLUSIVE;
- ExclusiveLock блокировки, Этот режим совместим только с блокировкой ACCESS SHARE, то есть параллельно с транзакцией, получившей блокировку в этом режиме, допускается только чтение таблицы.

Сессия 2:
```
locks=# BEGIN;
SELECT pg_backend_pid();
 pg_backend_pid
----------------
           2658
(1 row)
UPDATE accounts SET amount = amount + 1000.00 WHERE acc_no = 1;

locks=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 2658;

   locktype    |   relation    | virtxid |   xid   |       mode       | granted 
---------------+---------------+---------+---------+------------------+---------
 relation      | accounts_pkey |         |         | RowExclusiveLock | t
 relation      | accounts      |         |         | RowExclusiveLock | t
 virtualxid    |               | 5/7     |         | ExclusiveLock    | t
 transactionid |               |         | 1508645 | ShareLock        | f
 tuple         | accounts      |         |         | ExclusiveLock    | t
 transactionid |               |         | 1508646 | ExclusiveLock    | t
(6 rows)
```
Транзакция началась и ожидает завершения предыдущей для выполнения (granted = f).    
Сессия 3: 
```
locks=# BEGIN;
SELECT pg_backend_pid();
 pg_backend_pid
----------------
           2764
(1 row)

UPDATE accounts SET amount = amount + 10000.00 WHERE acc_no = 1;

locks=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 2764;

   locktype    |   relation    | virtxid |   xid   |       mode       | granted 
---------------+---------------+---------+---------+------------------+---------
 relation      | accounts_pkey |         |         | RowExclusiveLock | t
 relation      | accounts      |         |         | RowExclusiveLock | t
 virtualxid    |               | 6/2     |         | ExclusiveLock    | t
 tuple         | accounts      |         |         | ExclusiveLock    | f
 transactionid |               |         | 1508647 | ExclusiveLock    | t
(5 rows)

locks=*# SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for FROM pg_locks WHERE relation = 'accounts'::regclass; \
 locktype |       mode       | granted | pid  | wait_for 
----------+------------------+---------+------+----------
 relation | RowExclusiveLock | t       | 2764 | {2658}
 relation | RowExclusiveLock | t       | 2658 | {2608}
 relation | RowExclusiveLock | t       | 2608 | {}
 tuple    | ExclusiveLock    | f       | 2764 | {2658}
 tuple    | ExclusiveLock    | t       | 2658 | {2608}
(5 rows)
```
Наблюдаются блокировки: 

- RowExclusiveLock, блокировки возникли при апдейте строки в таблице accounts. Конфликтует с режимами блокировки SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE и ACCESS EXCLUSIVE;
- ExclusiveLock блокировки, с ссылкой на tuple. 

Можно посмотреть общую картину ожиданий: 
```
postgres=# SELECT pid, wait_event_type, wait_event, pg_blocking_pids(pid)
FROM pg_stat_activity
WHERE backend_type = 'client backend' ORDER BY pid;
 pid  | wait_event_type |  wait_event   | pg_blocking_pids
------+-----------------+---------------+------------------
 2608 | Client          | ClientRead    | {}
 2658 | Client          | ClientRead    | {}
 2764 | Lock            | transactionid | {2658}
 3911 |                 |               | {}
(4 rows)
```
Попеременно завершим транзакции коммитом и посмотрим как уходят блокировки:

Сессия 1: 
```
locks=!# commit;
```
Сессия 2:
```
...
UPDATE 1

locks=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 2658;
   locktype    |   relation    | virtxid |   xid   |       mode       | granted
---------------+---------------+---------+---------+------------------+---------
 relation      | pg_locks      |         |         | AccessShareLock  | t
 relation      | accounts_pkey |         |         | RowExclusiveLock | t
 relation      | accounts      |         |         | RowExclusiveLock | t
 virtualxid    |               | 5/7     |         | ExclusiveLock    | t
 transactionid |               |         | 1508646 | ExclusiveLock    | t
(5 rows)

locks=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 2764;
   locktype    |   relation    | virtxid |   xid   |       mode       | granted
---------------+---------------+---------+---------+------------------+---------
 relation      | accounts_pkey |         |         | RowExclusiveLock | t
 relation      | accounts      |         |         | RowExclusiveLock | t
 virtualxid    |               | 6/2     |         | ExclusiveLock    | t
 tuple         | accounts      |         |         | ExclusiveLock    | t
 transactionid |               |         | 1508646 | ShareLock        | f
 transactionid |               |         | 1508647 | ExclusiveLock    | t
(6 rows)

locks=*# commit;
COMMIT
```
Сессия 3:
```
...
UPDATE 1

locks=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 2764;
   locktype    |   relation    | virtxid |   xid   |       mode       | granted
---------------+---------------+---------+---------+------------------+---------
 relation      | pg_locks      |         |         | AccessShareLock  | t
 relation      | accounts_pkey |         |         | RowExclusiveLock | t
 relation      | accounts      |         |         | RowExclusiveLock | t
 virtualxid    |               | 6/2     |         | ExclusiveLock    | t
 transactionid |               |         | 1508647 | ExclusiveLock    | t
(5 rows)

locks=*# commit;
COMMIT
```
### Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
* Проверим значение параметров deadlock_timeout и lock_timeout:
```
locks=# SHOW deadlock_timeout;
SHOW lock_timeout;
 deadlock_timeout 
-----------------
 200ms
(1 row)

 lock_timeout 
--------------
 0
(1 row)
```
* Смоделируем вариант deadlocka 3-х транзакций:
Сессия 1:
```
locks=# BEGIN;
UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 1;
BEGIN
UPDATE 1
```
Сессия 2:
```
locks=# BEGIN;
UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 2;
BEGIN
UPDATE 1
```
Сессия 3:
```
locks=# BEGIN;
UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 3;
BEGIN
UPDATE 1
```
Сессия 1:
```
UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 2;
```
Сессия 2:
```
UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 3;
```
Сессия 3:
```
UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 1;
ERROR:  deadlock detected
DETAIL:  Process 4116 waits for ShareLock on transaction 1508651; blocked by process 2608.
Process 2608 waits for ShareLock on transaction 1508652; blocked by process 2764.
Process 2764 waits for ShareLock on transaction 1508653; blocked by process 4116.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,6) in relation "accounts"
```
Третья сессия была убита из-за дедлока, вторая сессия выполнила апдейт, первая сессия ожидает. 
В логах также можно найти информацию о дедлоке:
```
2023-07-19 17:05:31.698 MSK [2608] postgres@locks LOG:  process 2608 still waiting for ShareLock on transaction 1508652 after 201.092 ms
2023-07-19 17:05:31.698 MSK [2608] postgres@locks DETAIL:  Process holding the lock: 2764. Wait queue: 2608.
2023-07-19 17:05:31.698 MSK [2608] postgres@locks CONTEXT:  while updating tuple (0,2) in relation "accounts"
2023-07-19 17:05:31.698 MSK [2608] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 2;
2023-07-19 17:05:40.675 MSK [2764] postgres@locks LOG:  process 2764 still waiting for ShareLock on transaction 1508653 after 200.882 ms
2023-07-19 17:05:40.675 MSK [2764] postgres@locks DETAIL:  Process holding the lock: 4116. Wait queue: 2764.
2023-07-19 17:05:40.675 MSK [2764] postgres@locks CONTEXT:  while updating tuple (0,3) in relation "accounts"
2023-07-19 17:05:40.675 MSK [2764] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 3;

2023-07-19 17:05:48.322 MSK [4116] postgres@locks LOG:  process 4116 detected deadlock while waiting for ShareLock on transaction 1508651 after 200.284 ms
2023-07-19 17:05:48.322 MSK [4116] postgres@locks DETAIL:  Process holding the lock: 2608. Wait queue: .
2023-07-19 17:05:48.322 MSK [4116] postgres@locks CONTEXT:  while updating tuple (0,6) in relation "accounts"
2023-07-19 17:05:48.322 MSK [4116] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 1;
2023-07-19 17:05:48.322 MSK [4116] postgres@locks ERROR:  deadlock detected
2023-07-19 17:05:48.322 MSK [4116] postgres@locks DETAIL:  Process 4116 waits for ShareLock on transaction 1508651; blocked by process 2608.
        Process 2608 waits for ShareLock on transaction 1508652; blocked by process 2764.
        Process 2764 waits for ShareLock on transaction 1508653; blocked by process 4116.
        Process 4116: UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 1;
        Process 2608: UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 2;
        Process 2764: UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 3;

2023-07-19 17:05:48.322 MSK [4116] postgres@locks HINT:  See server log for query details.
2023-07-19 17:05:48.322 MSK [4116] postgres@locks CONTEXT:  while updating tuple (0,6) in relation "accounts"
2023-07-19 17:05:48.322 MSK [4116] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 1;
2023-07-19 17:05:48.323 MSK [2764] postgres@locks LOG:  process 2764 acquired ShareLock on transaction 1508653 after 7848.564 ms
2023-07-19 17:05:48.323 MSK [2764] postgres@locks CONTEXT:  while updating tuple (0,3) in relation "accounts"
2023-07-19 17:05:48.323 MSK [2764] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 3;
```
### Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
Мне не удалось смоделировать ситуацию взаимной блокировки (deadlocka) без where, варианты ниже.
### Задание со звездочкой*
### Попробуйте воспроизвести такую ситуацию.
```
locks=# Select * from accounts;
 acc_no |  amount
--------+----------
      1 | 11090.00
      3 |   290.00
      2 |   200.00
(3 rows)
```
Сессия 1 (начинает изменять строки столбца amount, заблокировала всю таблицу):
```
locks=# BEGIN;
SELECT pg_backend_pid();
UPDATE accounts SET amount = amount + 10.00;
BEGIN
UPDATE 3

 pg_backend_pid 
----------------
           4264
(1 row)
```
Сессия 2 (ожидает):
```
BEGIN;
SELECT pg_backend_pid();
UPDATE accounts SET amount = amount - 190.00;

 pg_backend_pid
----------------
           2764
(1 row)
```
Посмотрим блокировки данных транзакций: 
```
locks=# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 4264;
   locktype    |   relation    | virtxid |   xid   |       mode       | granted
---------------+---------------+---------+---------+------------------+---------
 relation      | accounts_pkey |         |         | RowExclusiveLock | t
 relation      | accounts      |         |         | RowExclusiveLock | t
 virtualxid    |               | 4/20    |         | ExclusiveLock    | t
 transactionid |               |         | 1508656 | ExclusiveLock    | t
(4 rows)

locks=# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 2764;
   locktype    |   relation    | virtxid |   xid   |       mode       | granted
---------------+---------------+---------+---------+------------------+---------
 relation      | accounts_pkey |         |         | RowExclusiveLock | t
 relation      | accounts      |         |         | RowExclusiveLock | t
 virtualxid    |               | 6/8     |         | ExclusiveLock    | t
 transactionid |               |         | 1508657 | ExclusiveLock    | t
 transactionid |               |         | 1508656 | ShareLock        | f
 tuple         | accounts      |         |         | ExclusiveLock    | t
(6 rows)
```
В данном случае первая транзакция выполнилась, а вторая находится в ожидании.


Также была попытка создать таблицу, в которой первая сессия будет менять данные из столбца amount, а вторая будет менять данные из столбца summ, и затем попытаться сделать взаимную блокировку, однако взаимной блокировки не случилось, т.к. 1 транзакция заблокировала полностью всю таблицу:
```
locks=# CREATE TABLE accounts_2(
  acc_no integer PRIMARY KEY,
  amount numeric,
  summ numeric
);

locks=# INSERT INTO accounts_2 VALUES (1, 100.00, 200.00), (2, 200.00, 300.00), (3, 300.00, 400.00);
INSERT 0 3
```
Сессия 1:
```
locks=# BEGIN;
UPDATE accounts_2 SET amount = amount + 10.00;
BEGIN
```
Сессия 2:
```
locks=# BEGIN;
UPDATE accounts_2 SET amount = summ + 10.00;
BEGIN
```
Сессия 1:
```
locks=*# UPDATE accounts_2 SET amount = summ - 100.00;
UPDATE 3
```
Сессия 2 (все еще в ожидании):
```
UPDATE accounts_2 SET amount = amount + 100.00;
```