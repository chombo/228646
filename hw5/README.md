# MVCC, vacuum и autovacuum

### Подготовка стенда
Задние выполняется на виртуальной машине из
https://github.com/chombo/228646/blob/main/hw3/README.md

```shell
postgres@test-Postgresql1:/$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
```shell
postgres@test-Postgresql1:/$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.22 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.50 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.32 s, vacuum 0.09 s, primary keys 0.08 s).
```
С настройками по умолчанию
```shell
postgres@test-Postgresql1:/$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.2 (Ubuntu 15.2-1.pgdg20.04+1))
starting vacuum...end.
progress: 6.0 s, 911.3 tps, lat 8.695 ms stddev 4.952, 0 failed
progress: 12.0 s, 842.8 tps, lat 9.460 ms stddev 5.507, 0 failed
progress: 18.0 s, 852.5 tps, lat 9.367 ms stddev 5.353, 0 failed
progress: 24.0 s, 863.8 tps, lat 9.243 ms stddev 5.572, 0 failed
progress: 30.0 s, 785.2 tps, lat 10.178 ms stddev 6.636, 0 failed
progress: 36.0 s, 695.2 tps, lat 11.483 ms stddev 7.302, 0 failed
progress: 42.0 s, 806.0 tps, lat 9.912 ms stddev 5.725, 0 failed
progress: 48.0 s, 869.4 tps, lat 9.201 ms stddev 5.579, 0 failed
progress: 54.0 s, 833.5 tps, lat 9.582 ms stddev 5.547, 0 failed
progress: 60.0 s, 873.3 tps, lat 9.139 ms stddev 5.276, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 50006
number of failed transactions: 0 (0.000%)
latency average = 9.577 ms
latency stddev = 5.780 ms
initial connection time = 45.411 ms
tps = 833.802889 (without initial connection time)
```
Меняем настройки на предложенные
```shell
postgres=# alter system set max_connections = '40';
ALTER SYSTEM
postgres=# alter system set shared_buffers = '1GB';
ALTER SYSTEM
postgres=# alter system set effective_cache_size = '3GB';
ALTER SYSTEM
postgres=# alter system set maintenance_work_mem = '512MB';
ALTER SYSTEM
postgres=# alter system set checkpoint_completion_target = '0.9';
ALTER SYSTEM
postgres=# alter system set wal_buffers = '16MB';
ALTER SYSTEM
postgres=# alter system set default_statistics_target = '500';
ALTER SYSTEM
postgres=# alter system set random_page_cost = '4';
ALTER SYSTEM
postgres=# alter system set effective_io_concurrency = '2';
ALTER SYSTEM
postgres=# alter system set work_mem = '6553kB';
ALTER SYSTEM
postgres=# alter system set min_wal_size = '4GB';
ALTER SYSTEM
postgres=# alter system set max_wal_size = '16GB';
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```
Рестарт кластера: pg_ctlcluster 15 main restart (часть значений требует рестарта)
```shell
postgres@test-Postgresql1:/$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.2 (Ubuntu 15.2-1.pgdg20.04+1))
starting vacuum...end.
progress: 6.0 s, 876.3 tps, lat 9.036 ms stddev 5.038, 0 failed
progress: 12.0 s, 832.8 tps, lat 9.578 ms stddev 5.470, 0 failed
progress: 18.0 s, 754.5 tps, lat 10.568 ms stddev 6.597, 0 failed
progress: 24.0 s, 804.5 tps, lat 9.914 ms stddev 5.923, 0 failed
progress: 30.0 s, 838.7 tps, lat 9.515 ms stddev 5.650, 0 failed
progress: 36.0 s, 847.7 tps, lat 9.409 ms stddev 5.628, 0 failed
progress: 42.0 s, 826.0 tps, lat 9.646 ms stddev 5.511, 0 failed
progress: 48.0 s, 822.5 tps, lat 9.697 ms stddev 5.771, 0 failed
progress: 54.0 s, 827.2 tps, lat 9.649 ms stddev 6.172, 0 failed
progress: 60.0 s, 899.0 tps, lat 8.877 ms stddev 5.151, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 49983
number of failed transactions: 0 (0.000%)
latency average = 9.570 ms
latency stddev = 5.707 ms
initial connection time = 45.634 ms
tps = 833.342253 (without initial connection time)
```
Устанавливаем shared_buffers 25% от объёма памяти (от 8GB)
```shell
postgres=# alter system set shared_buffers = '2GB';
```
```shell
pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.2 (Ubuntu 15.2-1.pgdg20.04+1))
starting vacuum...end.
progress: 6.0 s, 829.7 tps, lat 9.528 ms stddev 5.365, 0 failed
progress: 12.0 s, 850.5 tps, lat 9.388 ms stddev 5.411, 0 failed
progress: 18.0 s, 848.9 tps, lat 9.409 ms stddev 5.551, 0 failed
progress: 24.0 s, 841.6 tps, lat 9.488 ms stddev 5.715, 0 failed
progress: 30.0 s, 810.9 tps, lat 9.849 ms stddev 5.986, 0 failed
progress: 36.0 s, 835.2 tps, lat 9.564 ms stddev 5.697, 0 failed
progress: 42.0 s, 824.3 tps, lat 9.684 ms stddev 5.853, 0 failed
progress: 48.0 s, 838.2 tps, lat 9.525 ms stddev 5.638, 0 failed
progress: 54.0 s, 841.3 tps, lat 9.487 ms stddev 5.565, 0 failed
progress: 60.0 s, 829.0 tps, lat 9.613 ms stddev 5.949, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 50104
number of failed transactions: 0 (0.000%)
latency average = 9.553 ms
latency stddev = 5.677 ms
initial connection time = 54.007 ms
tps = 835.529174 (without initial connection time)
```
Что изменилось и почему?
default (8MB)
tps = 833.802889
предложенный (shared_buffers = '1GB')
tps = 833.342253
shared_buffers = '2GB'
tps = 835.529174

-c: 8 количество одновременных клиентов
-P: отображает прогресс и метрики каждые 6 секунд.
-T: запустит тест на 60 секунд

увеличение tps (в данном случае) - обусловленно изменением shared_buffers - чем больше данных к которым требуется доступ присутствуют в shared_buffers - тем быстрее к ним осуществляется доступ.



```sql
postgres=# CREATE DATABASE hw5;
postgres=# \c hw5
hw5=# CREATE TABLE hw5 (text_coll TEXT);
hw5=# INSERT INTO hw5(text_coll) SELECT md5(random()::text) FROM generate_series(1,1000000);
INSERT 0 1000000
```

```shell
hw5=# \dt+
 Schema | Name | Type  |  Owner   | Persistence | Access method |  Size  | Description
--------+------+-------+----------+-------------+---------------+--------+-------------
 public | hw5  | table | postgres | permanent   | heap          | 50 MB |

hw5=# UPDATE hw5 SET text_coll = text_coll || '1';
UPDATE 1000000
hw5=# UPDATE hw5 SET text_coll = text_coll || '2';
UPDATE 1000000
hw5=# UPDATE hw5 SET text_coll = text_coll || '3';
UPDATE 1000000
hw5=# UPDATE hw5 SET text_coll = text_coll || '4';
UPDATE 1000000
hw5=# UPDATE hw5 SET text_coll = text_coll || '5';
UPDATE 1000000
hw5=# \dt+
 Schema | Name | Type  |  Owner   | Persistence | Access method |  Size  | Description
--------+------+-------+----------+-------------+---------------+--------+-------------
 public | hw5  | table | postgres | permanent   | heap          | 254 MB |
```

```shell
last_autovacuum from pg_stat_all_tables WHERE relname = 'hw5';
        last_autovacuum
-------------------------------
 2023-05-09 20:11:48.032202+03
(1 row)
hw5=# UPDATE hw5 SET text_coll = text_coll || '6';
UPDATE 1000000
hw5=# UPDATE hw5 SET text_coll = text_coll || '7';
UPDATE 1000000
hw5=# UPDATE hw5 SET text_coll = text_coll || '8';
UPDATE 1000000
hw5=# UPDATE hw5 SET text_coll = text_coll || '9';
UPDATE 1000000
hw5=# UPDATE hw5 SET text_coll = text_coll || '0';
UPDATE 1000000
hw5=# \dt+
                                   List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method |  Size  | Description
--------+------+-------+----------+-------------+---------------+--------+-------------
 public | hw5  | table | postgres | permanent   | heap          | 359 MB |
hw5=# select last_autovacuum from pg_stat_all_tables WHERE relname = 'hw5';
        last_autovacuum
-------------------------------
 2023-05-09 20:23:48.627086+03
(1 row)
hw5=# \dt+
                                   List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method |  Size  | Description
--------+------+-------+----------+-------------+---------------+--------+-------------
 public | hw5  | table | postgres | permanent   | heap          | 359 MB |
```

```shell
hw5=# ALTER TABLE hw5 SET (autovacuum_enabled = off);
ALTER TABLE
hw5=# call update_data_ten_times ('hw5');
...
hw5=# select last_autovacuum from pg_stat_all_tables WHERE relname = 'hw5';
        last_autovacuum
-------------------------------
 2023-05-09 20:23:48.627086+03
(1 row)

hw5=# \dt+
                                   List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method |  Size  | Description
--------+------+-------+----------+-------------+---------------+--------+-------------
 public | hw5  | table | postgres | permanent   | heap          | 571 MB |
hw5=# ALTER TABLE hw5 SET (autovacuum_enabled = on);
ALTER TABLE
```
autovacuum - не возвращает память
```shell
hw5=# vacuum full hw5;
VACUUM
hw5=# \dt+
                                  List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method | Size  | Description
--------+------+-------+----------+-------------+---------------+-------+-------------
 public | hw5  | table | postgres | permanent   | heap          | 81 MB |
```


### Задание со *:
Написать анонимную процедуру, в которой в цикле 10 раз
обновятся все строчки в искомой таблице.
Не забыть вывести номер шага цикла.

```sql
CREATE OR REPLACE PROCEDURE update_data_ten_times (_tbl regclass) AS $$
DECLARE
a INT:= 10;
BEGIN
RAISE INFO 'Вызов функции update_data_ten_times (%)', _tbl;
FOR i IN 1..a LOOP
RAISE INFO 'i:(%)', i;
EXECUTE format('UPDATE %s SET text_coll = ''(md5(random()::text)''', _tbl);
END LOOP;
END;
$$
LANGUAGE plpgsql;
CREATE PROCEDURE
```

```sql
hw5=# CALL update_data_ten_times('hw1');
INFO:  Вызов функции update_data_ten_times (hw1)
INFO:  i:(1)
INFO:  i:(2)
INFO:  i:(3)
INFO:  i:(4)
INFO:  i:(5)
INFO:  i:(6)
INFO:  i:(7)
INFO:  i:(8)
INFO:  i:(9)
INFO:  i:(10)
CALL
```
