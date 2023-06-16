# Домашнее задание "Тема 3. Установка PostgreSQL":

* __Cоздана ВМ с Ubuntu 22.04__
```
root@ubuntu:/var/lib/postgres# cat /etc/os-release
PRETTY_NAME="Ubuntu 22.04.2 LTS"
```
* __Установлен на ней Docker Engine__
```
tatiana@ubuntu:~$ sudo su -
root@ubuntu:~# apt-get update
root@ubuntu:~# apt install curl
root@ubuntu:~# curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER
```
***Создана docker-сеть***:
```
root@ubuntu:~# sudo docker network create pg-net
```
* __Создан каталог /var/lib/postgres__
* __Развернут контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql__
```
root@ubuntu:~# sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15

Unable to find image 'postgres:15' locally
15: Pulling from library/postgres
5b5fe70539cd: Pull complete 
254f758dbb83: Pull complete 
a50b6f601ef0: Pull complete 
b2a96a17b000: Pull complete 
b08a8624844f: Pull complete 
9fb1f8c3deef: Pull complete 
fb374f351a53: Pull complete 
7b3d9f705bf0: Pull complete 
a85d8c4f987c: Pull complete 
0543b0f5d286: Pull complete 
28bb5ab88d7a: Pull complete 
fdb6f3d907fc: Pull complete 
f06aa6280d55: Pull complete 
Digest: sha256:7c0ee16b6a3b4403957ece2c186ff05c57097a557403ae5216ef1286e47c249c
Status: Downloaded newer image for postgres:15
e0de28938b13a72de8b8c0c7b2e9dfa0d8902b6215f552f08a85807a9b5a0d2c
```
* __Развернут контейнер с клиентом postgres__
```
root@ubuntu:~# sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
Password for user postgres: 
psql (15.3 (Debian 15.3-1.pgdg120+1))
```
* __Выполнено подключение из контейнера с клиентом к контейнеру с сервером и создана таблицу с несколькими строками__
```
root@ubuntu:/var/lib/postgres# sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
e0de28938b13   postgres:15   "docker-entrypoint.s…"   51 minutes ago   Up 32 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server

postgres=# create database docker_test;
postgres=# CREATE TABLE accounts (
	user_id serial PRIMARY KEY,
	username VARCHAR ( 50 ) UNIQUE NOT NULL,
	password VARCHAR ( 50 ) NOT NULL,
	email VARCHAR ( 255 ) UNIQUE NOT NULL
);
postgres=# INSERT INTO accounts(user_id, username, password, email)
VALUES (1, 'Ivan', 'Pass', 'Ivan@mail.ru');
postgres=# INSERT INTO accounts(user_id, username, password, email)
VALUES (2, 'Poman', 'Pass', 'Poman@mail.ru');
postgres=# INSERT INTO accounts(user_id, username, password, email)
VALUES (3, 'Bogdan', 'Pass', 'Bogdan@mail.ru');
```
* __Выполнено подключение к контейнеру с сервером с ноутбука извне инстансов ВМ__
```
login as: tatiana
tatiana@192.168.56.102's password:

tatiana@ubuntu:~$ sudo docker exec -it pg-server bash

root@e0de28938b13:/# psql -U postgres
psql (15.3 (Debian 15.3-1.pgdg120+1))
Type "help" for help.

postgres=# select * from accounts;
 user_id | username | password |     email
---------+----------+----------+----------------
       1 | Ivan     | Pass     | Ivan@mail.ru
       2 | Poman    | Pass     | Poman@mail.ru
       3 | Bogdan   | Pass     | Bogdan@mail.ru
(3 rows)
```
* __Удален контейнер с сервером__
```
root@ubuntu:/var/lib/postgres# sudo docker stop e0de28938b13
e0de28938b13

root@ubuntu:/var/lib/postgres# docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

root@ubuntu:/var/lib/postgres# sudo docker rm e0de28938b13
e0de28938b13
```
* __Контейнер создан заново__
```
root@ubuntu:/var/lib/postgres# sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
2c7038877c029d59f625f98915af790379b0267348331504a0dd1d197da7605f

root@ubuntu:/var/lib/postgres# docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
2c7038877c02   postgres:15   "docker-entrypoint.s…"   32 seconds ago   Up 32 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
```
* __Выполнено подключение снова из контейнера с клиентом к контейнеру с сервером__
```
root@ubuntu:/var/lib/postgres# sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
Password for user postgres: 
psql (15.3 (Debian 15.3-1.pgdg120+1))
```
* __Выполнена проверка, что данные остались на месте__
```
postgres=# \l
                                                 List of databases
    Name     |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges   

-------------+----------+----------+------------+------------+------------+-----------------+-----------------------
 docker_test | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | 
 postgres    | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | 
 template0   | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
             |          |          |            |            |            |                 | postgres=CTc/postgres
 template1   | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
             |          |          |            |            |            |                 | postgres=CTc/postgres
(4 rows)

postgres=# select * from accounts;
 user_id | username | password |     email      
---------+----------+----------+----------------
       1 | Ivan     | Pass     | Ivan@mail.ru
       2 | Poman    | Pass     | Poman@mail.ru
       3 | Bogdan   | Pass     | Bogdan@mail.ru
(3 rows)
```
