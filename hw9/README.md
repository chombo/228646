# Резервное копирование и восстановление

```shell
docker exec -ti -u postgres postgres_1 sh
psql
```
```sql
postgres=# CREATE DATABASE hw10;
CREATE DATABASE
postgres=# \c hw10
hw10=# CREATE s
schema        sequence      server        statistics    subscription 
hw10=# CREATE schema hw10;
CREATE SCHEMA
hw10=# create table hw10.t1(id serial, message text);
CREATE TABLE
hw10=# INSERT INTO hw10.t1(message) SELECT md5(random()::text) FROM generate_series(1,100);
INSERT 0 100
```

```shell
/ # mkdir /backups
/ # ls -al /
...
drwxr-xr-x    2 postgres postgres      4096 Jun 30 05:34 backups
...
```

```sql
postgres=# \c hw10;
You are now connected to database "hw10" as user "postgres".
hw10=# \copy hw10.t1 to '/backups/30062023_1.sql';
COPY 100
hw10=# create table hw10.t2(id serial, message text);
CREATE TABLE
hw10=# \copy hw10.t2 from '/backups/30062023_1.sql';
COPY 100
```

```shell
pg_dump -d hw10 -Fc > /backups/30062023_2.gz
```

```sql
postgres=# CREATE DATABASE hw10_v2;
CREATE DATABASE
postgres=# \c hw10_v2
hw10_v2=# CREATE schema hw10;
```

```shell
pg_restore -d hw10_v2 --table=t2 /backups/30062023_2.gz 
```

```sql
postgres=# \c  hw10_v2
hw10_v2=# SELECT FROM hw10.t2;
--
(100 rows)

```
