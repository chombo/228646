# SQL и реляционные СУБД. Введение в PostgreSQL

### Подготовка стенда

Поднимаем контейнер с сервером postgres:
```shell
docker pull postgres:15.2-alpine3.17
docker network create nw-postgres
docker run -d --rm --name postgres_1 -e POSTGRES_PASSWORD=postgres \
--network nw-postgres \
-v ~/hw1/tmp/data:/var/lib/postgresql/data postgres:15.2-alpine3.17
```

```shell
docker ps
CONTAINER ID   IMAGE                      COMMAND                  CREATED         STATUS         PORTS      NAMES
505024605f92   postgres:15.2-alpine3.17   "docker-entrypoint.s…"   5 seconds ago   Up 4 seconds   5432/tcp   postgres_1
```

Подключиться к поднятому серверу из 2х терминалов, используя команду:
```shell
docker run --rm -ti --network nw-postgres postgres:15.2-alpine3.17 psql -h postgres_1 -U postgres            
Password for user postgres: 
psql (15.2)
```
Создаем базу данных для тестов:
```shell
postgres=# CREATE DATABASE hw1;
CREATE DATABASE
postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges   
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 hw1       | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | 
postgres=# \c hw1
You are now connected to database "hw1" as user "postgres".
```

В первом терминале:

```shell
hw1=# \echo :AUTOCOMMIT
on
hw1=# \set AUTOCOMMIT OFF
hw1=# \echo :AUTOCOMMIT
OFF
hw1=# create table persons(id serial, first_name text, second_name text);
CREATE TABLE
hw1=*#  insert into persons(first_name, second_name) values('ivan', 'ivanov');
INSERT 0 1
hw1=*#  insert into persons(first_name, second_name) values('petr', 'petrov');
INSERT 0 1
hw1=*#  commit;
COMMIT
```

### 1) read committed

Смотрим текущий уровень изоляции:
```shell
hw1=# show transaction isolation level;
 transaction_isolation 
-----------------------
 read committed
(1 row)
```

В первом терминале:
```shell
hw1=# START TRANSACTION;
START TRANSACTION
hw1=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
hw1=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

```

Во втором терминале:
```shell
hw1=# START TRANSACTION;
START TRANSACTION
hw1=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
**ОТВЕТ:**
_транзакция в первом терминале еще не завершена - запись в таблице не отображается._

**ПОЯСНЕНИЕ:** _Транзакции — это фундаментальное понятие во всех СУБД.
Суть транзакции в том, что она объединяет последовательность действий 
в одну операцию «всё или ничего». Промежуточные состояния внутри 
последовательности не видны другим транзакциям, и если что-то 
помешает успешно завершить транзакцию, ни один из результатов этих 
действий не сохранится в базе данных._

Завершить транзакцию в первом терминале:
```shell
hw1=# hw1=*# COMMIT;
COMMIT
```

Во втором терминале (теперь запись отображается):
```shell
hw1=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

```
### 2) repeatable read
Установить repeatable read, в обоих терминалах:

```shell
hw1=# set transaction isolation level repeatable read;
SET
hw1=*# show transaction isolation level;
 transaction_isolation 
-----------------------
 repeatable read
(1 row)

```

Выполним в первом:
```shell
hw1=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```

Выполним во втором:
```shell
hw1=*#  select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

Завершаем транзакцию в первом терминале:

```shell
hw1=*# commit;
COMMIT
hw1=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

Смотрим во втором:

```shell
hw1=*#  select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

**ОТВЕТ:**
_Добавленная строка, все еще не отображается._

**ПОЯСНЕНИЕ:** _В данном случае во второй транзакции (до завершения первой 
транзации) был получен набор данных. Вторая транзакий работает со своей копией данных.s_ 
 
Выполним во втором терминале:

```shell
hw1=*# commit;
COMMIT
hw1=#  select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)

```
Если выполнить commit в первом терминале до select во втором,
то select до завершения commit во втором вернет новый набор строк. 





