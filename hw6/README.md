# Блокировки 

### Подготовка стенда

```shell
docker pull postgres:15.2-alpine3.17
docker network create nw-postgres
docker run  --rm --name postgres_1 -e POSTGRES_PASSWORD=postgres \
--network nw-postgres \
-v $(pwd)/hw6/tmp/data:/var/lib/postgresql/data postgres:15.2-alpine3.17
```

```sql
postgres=# show deadlock_timeout;
 deadlock_timeout 
------------------
 1s
(1 row)

postgres=# show log_lock_waits;
 log_lock_waits 
----------------
 off
(1 row)
postgres=# ALTER SYSTEM SET deadlock_timeout TO 200;
ALTER SYSTEM
postgres=# select pg_reload_conf();
pg_reload_conf
----------------
t
(1 row)

postgres=# show deadlock_timeout;
 deadlock_timeout 
------------------
200ms
(1 row)

postgres=# ALTER SYSTEM SET log_lock_waits TO on;
ALTER SYSTEM
postgres=# select pg_reload_conf();
pg_reload_conf
----------------
t
(1 row)

postgres=# show log_lock_waits;
 log_lock_waits 
----------------
on
(1 row)
```

```sql
postgres=# CREATE DATABASE hw6;
CREATE DATABASE
postgres=# \c hw6
You are now connected to database "hw6" as user "postgres".
hw6=# CREATE TABLE hw6(id serial, message text);
CREATE TABLE
hw6=# INSERT INTO hw6 values (1, 'message1');
INSERT 0 1
hw6=# INSERT INTO hw6 values (2, 'message2');
INSERT 0 1
```
### №1
session 1 
```sql
hw6=# BEGIN;
BEGIN
hw6=*# SELECT message FROM hw6 WHERE id = 1 FOR UPDATE;
message
-------------------
message2 update 2
(1 row)
```
session 2
```sql
hw6=# UPDATE hw6 SET message = 'message2 update 2' WHERE id = 1;
```
LOG
```shell
2023-06-04 15:23:25.159 UTC [134] LOG:  process 134 still waiting for ShareLock on transaction 758 after 201.618 ms
2023-06-04 15:23:25.159 UTC [134] DETAIL:  Process holding the lock: 86. Wait queue: 134.
2023-06-04 15:23:25.159 UTC [134] CONTEXT:  while updating tuple (0,7) in relation "hw6"
2023-06-04 15:23:25.159 UTC [134] STATEMENT:  UPDATE hw6 SET message = 'message2 update 2' WHERE id = 1;
2023-06-04 15:26:46.280 UTC [134] LOG:  process 134 acquired ShareLock on transaction 758 after 201321.928 ms
2023-06-04 15:26:46.280 UTC [134] CONTEXT:  while updating tuple (0,7) in relation "hw6"
2023-06-04 15:26:46.280 UTC [134] STATEMENT:  UPDATE hw6 SET message = 'message2 update 2' WHERE id = 1;
```
## №2

session 1
```sql
hw6=# BEGIN;
BEGIN
hw6=*# SELECT message FROM hw6 WHERE id = 1 FOR UPDATE;
message
-------------------
message2 update 2
(1 row)
```

session 2
```sql
hw6=# BEGIN;
BEGIN
hw6=*# SELECT message FROM hw6 WHERE id = 1 FOR UPDATE;
message
-------------------
message2 update 2
             (1 row)
```

session 1
```sql
hw6=# BEGIN;
BEGIN
hw6=*# SELECT message FROM hw6 WHERE id = 1 FOR UPDATE;
message
-------------------
message2 update 2
             (1 row)
```

## №3

Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны.
Пришлите список блокировок и объясните, что значит каждая.

session 1 9702
```sql
hw6=# BEGIN;
BEGIN
hw6=*# UPDATE hw6 SET message = 'session #1' WHERE id = 1;
UPDATE 1
```
```sql
   locktype    | relation |       mode       | tid | vtid | pid  | granted 
---------------+----------+------------------+-----+------+------+---------
 relation      | 32770    | RowExclusiveLock |     | 10/5 | 9702 | t
 virtualxid    |          | ExclusiveLock    |     | 10/5 | 9702 | t
 transactionid |          | ExclusiveLock    | 770 | 10/5 | 9702 | t
(3 rows)
```
session 2 9713
```sql
hw6=# BEGIN;
BEGIN
hw6=*# UPDATE hw6 SET message = 'session #2' WHERE id = 1;
```
```sql
   locktype    | relation |       mode       | tid | vtid | pid  | granted 
---------------+----------+------------------+-----+------+------+---------
 virtualxid    |          | ExclusiveLock    |     | 10/5 | 9702 | t
 relation      | 32770    | RowExclusiveLock |     | 10/5 | 9702 | t
 transactionid |          | ExclusiveLock    | 770 | 10/5 | 9702 | t
 transactionid |          | ExclusiveLock    | 771 | 11/5 | 9713 | t
 transactionid |          | ShareLock        | 770 | 11/5 | 9713 | f
 tuple         | 32770    | ExclusiveLock    |     | 11/5 | 9713 | t
 virtualxid    |          | ExclusiveLock    |     | 11/5 | 9713 | t
 relation      | 32770    | RowExclusiveLock |     | 11/5 | 9713 | t
(8 rows)
```
session 3 9722
```sql
hw6=# BEGIN;
BEGIN
hw6=*# UPDATE hw6 SET message = 'session #3' WHERE id = 1;
```
```sql
postgres=# select
               lock.locktype,
               lock.relation,
                   lock.mode,
               lock.transactionid as tid,
               lock.virtualtransaction as vtid,
               lock.pid,
               lock.granted
           from pg_catalog.pg_locks lock
                    left join pg_catalog.pg_database db
                              on db.oid = lock.database
           where (db.datname = 'hw6' or db.datname is null)
             and not lock.pid = pg_backend_pid()
           order by lock.pid;
  locktype       | relation |       mode       | tid | vtid | pid  | granted 
---------------+----------+------------------+-----+------+------+---------
   transactionid |          | ExclusiveLock    | 770 | 10/5 | 9702 | t
   relation      | 32770    | RowExclusiveLock |     | 10/5 | 9702 | t
   virtualxid    |          | ExclusiveLock    |     | 10/5 | 9702 | t
   tuple         | 32770    | ExclusiveLock    |     | 11/5 | 9713 | t
   relation      | 32770    | RowExclusiveLock |     | 11/5 | 9713 | t
   virtualxid    |          | ExclusiveLock    |     | 11/5 | 9713 | t
   transactionid |          | ExclusiveLock    | 771 | 11/5 | 9713 | t
   transactionid |          | ShareLock        | 770 | 11/5 | 9713 | f
   transactionid |          | ExclusiveLock    | 772 | 12/4 | 9722 | t
   virtualxid    |          | ExclusiveLock    |     | 12/4 | 9722 | t
   relation      | 32770    | RowExclusiveLock |     | 12/4 | 9722 | t
   tuple         | 32770    | ExclusiveLock    |     | 12/4 | 9722 | f
(12 rows)

```

1,3,6,7,9 - получены ExclusiveLock идентификаторы транзакции и виртуальной транзакции
8 - не получена ShareLock (на запись), ждет пока блокировка ExclusiveLock транзакции 770 не будет снята
2,5,11 - получены RowExclusiveLock на строку в таблице
4 - получена ExclusiveLock блокировка кортежа
12 - не получена ExclusiveLock кортежа (ждет снятия ExclusiveLock на tuple процесса 9713  )

## *****
Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

Выполням в 3х терминалах по шагам
setep 1: BEGIN;
setep 2: SELECT * FROM hw6 WHERE id = 1 FOR UPDATE;
setep 3: SELECT * FROM hw6 WHERE id = 2 FOR UPDATE;

setep 1: BEGIN;
setep 2: SELECT * FROM hw6 WHERE id = 2 FOR UPDATE;
setep 3: SELECT * FROM hw6 WHERE id = 3 FOR UPDATE;

setep 1: BEGIN;
setep 2: SELECT * FROM hw6 WHERE id = 3 FOR UPDATE;
setep 3: SELECT * FROM hw6 WHERE id = 1 FOR UPDATE;

В третьем получаем:
hw6=*# SELECT * FROM hw6 WHERE id = 1 FOR UPDATE;
ERROR:  deadlock detected
DETAIL:  Process 80 waits for ShareLock on transaction 737; blocked by process 62.
Process 62 waits for ShareLock on transaction 738; blocked by process 72.
Process 72 waits for ShareLock on transaction 739; blocked by process 80.
HINT:  See server log for query details.
CONTEXT:  while locking tuple (0,1) in relation "hw6"

```
2023-08-17 14:56:30.656 UTC [62] LOG:  process 62 still waiting for ShareLock on transaction 738 after 203.240 ms
2023-08-17 14:56:30.656 UTC [62] DETAIL:  Process holding the lock: 72. Wait queue: 62.
2023-08-17 14:56:30.656 UTC [62] CONTEXT:  while locking tuple (0,2) in relation "hw6"
2023-08-17 14:56:30.656 UTC [62] STATEMENT:  SELECT * FROM hw6 WHERE id = 2 FOR UPDATE;
2023-08-17 14:56:41.890 UTC [72] LOG:  process 72 still waiting for ShareLock on transaction 739 after 205.063 ms
2023-08-17 14:56:41.890 UTC [72] DETAIL:  Process holding the lock: 80. Wait queue: 72.
2023-08-17 14:56:41.890 UTC [72] CONTEXT:  while locking tuple (0,3) in relation "hw6"
2023-08-17 14:56:41.890 UTC [72] STATEMENT:  SELECT * FROM hw6 WHERE id = 3 FOR UPDATE;
2023-08-17 14:57:18.810 UTC [80] LOG:  process 80 detected deadlock while waiting for ShareLock on transaction 737 after 201.903 ms
2023-08-17 14:57:18.810 UTC [80] DETAIL:  Process holding the lock: 62. Wait queue: .
2023-08-17 14:57:18.810 UTC [80] CONTEXT:  while locking tuple (0,1) in relation "hw6"
2023-08-17 14:57:18.810 UTC [80] STATEMENT:  SELECT * FROM hw6 WHERE id = 1 FOR UPDATE;
2023-08-17 14:57:18.810 UTC [80] ERROR:  deadlock detected
2023-08-17 14:57:18.810 UTC [80] DETAIL:  Process 80 waits for ShareLock on transaction 737; blocked by process 62.
        Process 62 waits for ShareLock on transaction 738; blocked by process 72.
        Process 72 waits for ShareLock on transaction 739; blocked by process 80.
        Process 80: SELECT * FROM hw6 WHERE id = 1 FOR UPDATE;
        Process 62: SELECT * FROM hw6 WHERE id = 2 FOR UPDATE;
        Process 72: SELECT * FROM hw6 WHERE id = 3 FOR UPDATE;
2023-08-17 14:57:18.810 UTC [80] HINT:  See server log for query details.
2023-08-17 14:57:18.810 UTC [80] CONTEXT:  while locking tuple (0,1) in relation "hw6"
2023-08-17 14:57:18.810 UTC [80] STATEMENT:  SELECT * FROM hw6 WHERE id = 1 FOR UPDATE;
2023-08-17 14:57:18.811 UTC [72] LOG:  process 72 acquired ShareLock on transaction 739 after 37126.745 ms
2023-08-17 14:57:18.811 UTC [72] CONTEXT:  while locking tuple (0,3) in relation "hw6"
2023-08-17 14:57:18.811 UTC [72] STATEMENT:  SELECT * FROM hw6 WHERE id = 3 FOR UPDATE;
2023-08-17 14:59:23.613 UTC [49] LOG:  checkpoint starting: time
2023-08-17 14:59:23.781 UTC [49] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.111 s, sync=0.003 s, total=0.169 s; sync files=2, longest=0.003 s, average=0.002 s; distance=1 kB, estimate=3667 kB
```

Из лога винды id процессов и транзакций (и запрос в транзации) которые вызвали ERROR:  deadlock detected

## *****
Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
Да, возможно. Так как UPDATE — эта команда блокирует строки по мере их обновления, при разном порядке обновления возмлжна взаиная блокировка.
