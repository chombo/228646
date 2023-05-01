# Установка и настройка PostgteSQL в контейнере Docker

### Подготовка стенда

Поднимаем контейнер с сервером postgres:
```shell
docker pull postgres:15.2-alpine3.17
docker network create nw-postgres
docker run -d --rm --name postgres_1 -e POSTGRES_PASSWORD=postgres \
--network nw-postgres \
-p 5432:5432 \
-v ~/hw2/tmp/data:/var/lib/postgresql/data postgres:15.2-alpine3.17
```

```shell
docker ps
CONTAINER ID   IMAGE                      COMMAND                  CREATED         STATUS         PORTS                    NAMES
9fbfef92247f   postgres:15.2-alpine3.17   "docker-entrypoint.s…"   5 seconds ago   Up 4 seconds   0.0.0.0:5432->5432/tcp   postgres_1
```
```shell
docker port postgres_1
5432/tcp -> 0.0.0.0:5432
```
Подключиться к поднятому серверу:
```shell
docker run --rm -ti --network nw-postgres postgres:15.2-alpine3.17 psql -h postgres_1 -U postgres            
Password for user postgres: 
psql (15.2)
```

Создаем базу данных для тестов:
```shell
postgres=# CREATE DATABASE hw2;
CREATE DATABASE
postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges   
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 hw2       | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | 
postgres=# \c hw2
You are now connected to database "hw2" as user "postgres".
hw2=# create table test(id serial, name text);
CREATE TABLE
hw2=#  insert into test(name) values('name1');
INSERT 0 1
hw2=# SELECT * FROM test;
 id | name  
----+-------
  1 | name1
(1 row)
\q
```

Останавливаем и удаляем котейнер (удалится автоматически _--rm_)

```shell
docker kill postgres_1 
docker ps -a             
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```
Создаем заново и проверяем что данные сохранились:

```shell
docker run -d --rm --name postgres_1 -e POSTGRES_PASSWORD=postgres \
--network nw-postgres \
-p 5432:5432 \
-v ~/hw2/tmp/data:/var/lib/postgresql/data postgres:15.2-alpine3.17
```
```shell
docker run --rm --network nw-postgres postgres:15.2-alpine3.17 psql postgresql://postgres:postgres@postgres_1/hw2 -c 'SELECT * FROM test;'
 id | name  
----+-------
  1 | name1
(1 row)

```
```shell
ipconfig getifaddr en0
192.168.88.150
```
С другой машины в той же сети
```shell
telnet 192.168.88.150 5432
Trying 192.168.88.150...
Connected to 192.168.88.150.
Escape character is '^]'.
```

```shell
docker run --rm postgres:15.2-alpine3.17 psql postgresql://postgres:postgres@192.168.88.150/hw2 -c 'SELECT * FROM test;'
 id | name
----+-------
  1 | name1
(1 row)
```


