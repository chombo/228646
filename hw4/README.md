# Логический уровень PostgreSQL

### Подготовка стенда

Поднимаем контейнер с сервером postgres:
```shell
docker pull postgres:15.2-alpine3.17
docker network create nw-postgres
docker run -d --rm --name postgres_1 -e POSTGRES_PASSWORD=postgres \
--network nw-postgres \
-v ~/hw1/tmp/data:/var/lib/postgresql/data postgres:15.2-alpine3.17
```

Подключиться к поднятому серверу:
```shell
docker run --rm -ti --network nw-postgres postgres:15.2-alpine3.17 psql -h postgres_1 -U postgres            
Password for user postgres: 
psql (15.2)
```

### Ход выполнения ДЗ

```sql
postgres=# CREATE DATABASE testdb;
CREATE DATABASE
postgres=# \c testdb;
You are now connected to database "testdb" as user "postgres".
testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA
testdb=# CREATE TABLE t1(c1 int);
CREATE TABLE
testdb=# INSERT INTO t1(c1) values(1);
INSERT 0 1
testdb=# CREATE role readonly;
CREATE ROLE
testdb=# grant connect on DATABASE testdb TO readonly;
GRANT
testdb=# grant usage on SCHEMA testnm to readonly;
GRANT
testdb=# grant SELECT on all TABLES in SCHEMA testnm TO readonly;
GRANT
testdb=# CREATE USER testread with password 'test123';
CREATE ROLE
testdb=# grant readonly TO testread;
GRANT ROLE
testdb=# \c testdb testread
Password for user testread: 
You are now connected to database "testdb" as user "testread".
testdb=> select * from t1;
ERROR:  permission denied for table t1
```
t1 принадлежит схеме public (см ниже). Права на схему testnm.
В задании нет пояснеия в какой схеме создавать таблицу.
search_path - **"$user", public"**
```sql
testdb=# SHOW search_path;
   search_path   
-----------------
"$user", public
(1 row)
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
testdb=>  \dn
      List of schemas
  Name  |       Owner       
--------+-------------------
 public | pg_database_owner
 testnm | postgres
(2 rows)
```

```sql
testdb=# DROP TABLE t1;
DROP TABLE
testdb=# CREATE TABLE testnm.t1(c1 int);
CREATE TABLE
testdb=# INSERT INTO testnm.t1(c1) values(1);
INSERT 0 1
\c testdb testread;
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
Объект создан после _grant_.

```sql
testdb=# SELECT *
FROM information_schema.role_table_grants
WHERE table_name='t1';
 grantor  | grantee  | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy 
----------+----------+---------------+--------------+------------+----------------+--------------+----------------
 postgres | postgres | testdb        | testnm       | t1         | INSERT         | YES          | NO
 postgres | postgres | testdb        | testnm       | t1         | SELECT         | YES          | YES
 postgres | postgres | testdb        | testnm       | t1         | UPDATE         | YES          | NO
 postgres | postgres | testdb        | testnm       | t1         | DELETE         | YES          | NO
 postgres | postgres | testdb        | testnm       | t1         | TRUNCATE       | YES          | NO
 postgres | postgres | testdb        | testnm       | t1         | REFERENCES     | YES          | NO
 postgres | postgres | testdb        | testnm       | t1         | TRIGGER        | YES          | NO
(7 rows)
```
Добавим
```sql
testdb=# grant SELECT on all TABLES in SCHEMA testnm TO readonly;
GRANT
testdb=# SELECT *
FROM information_schema.role_table_grants
WHERE table_name='t1';
 grantor  | grantee  | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy 
----------+----------+---------------+--------------+------------+----------------+--------------+----------------
 postgres | postgres | testdb        | testnm       | t1         | INSERT         | YES          | NO
 postgres | postgres | testdb        | testnm       | t1         | SELECT         | YES          | YES
 postgres | postgres | testdb        | testnm       | t1         | UPDATE         | YES          | NO
 postgres | postgres | testdb        | testnm       | t1         | DELETE         | YES          | NO
 postgres | postgres | testdb        | testnm       | t1         | TRUNCATE       | YES          | NO
 postgres | postgres | testdb        | testnm       | t1         | REFERENCES     | YES          | NO
 postgres | postgres | testdb        | testnm       | t1         | TRIGGER        | YES          | NO
 postgres | readonly | testdb        | testnm       | t1         | SELECT         | NO           | YES
(8 rows)
```
Проверим
```sql
testdb=# \c testdb testread;
Password for user testread: 
You are now connected to database "testdb" as user "testread".
testdb=>  select * from testnm.t1;
 c1 
----
  1
(1 row)
```
[ALTER DEFAULT PRIVILEGES](https://postgrespro.ru/docs/postgrespro/15/sql-alterdefaultprivileges)

```sql
testdb=# ALTER DEFAULT PRIVILEGES IN SCHEMA testnm grant SELECT on TABLES to readonly;
ALTER DEFAULT PRIVILEGES
testdb=# CREATE TABLE testnm.t2(c1 int);
CREATE TABLE
testdb=# \c testdb testread;
Password for user testread: 
You are now connected to database "testdb" as user "testread".
testdb=> select * from testnm.t2;
 c1 
----
(0 rows)
```
```sql
testdb=> create table t2(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema public
testdb=> create table testnm.t2(c1 integer); 
ERROR:  permission denied for schema testnm
```

[роль PUBLIC](https://postgrespro.ru/docs/postgresql/12/ddl-priv#:~:text=PostgreSQL%20%D0%BF%D0%BE%20%D1%83%D0%BC%D0%BE%D0%BB%D1%87%D0%B0%D0%BD%D0%B8%D1%8E%20%D0%BD%D0%B0%D0%B7%D0%BD%D0%B0%D1%87%D0%B0%D0%B5%D1%82%20%D1%80%D0%BE%D0%BB%D0%B8,%D1%83%D0%BC%D0%BE%D0%BB%D1%87%D0%B0%D0%BD%D0%B8%D1%8E%20%D0%BD%D0%B8%D0%BA%D0%B0%D0%BA%D0%B8%D1%85%20%D0%BF%D1%80%D0%B0%D0%B2%20%D0%BD%D0%B5%20%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B0%D0%B5%D1%82.)
