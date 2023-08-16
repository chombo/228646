# Виды и устройство репликации в PostgreSQL.

поднимаем 4 ВМ с postgres 15

На ВМ1 и ВМ2
```
psql -c "ALTER SYSTEM SET wal_level = logical;"
```
рестартим postgres

ВМ1
```sql
postgres=# CREATE DATABASE host1;
CREATE DATABASE
postgres=# \c host1
You are now connected to database "host1" as user "postgres".
host1=# CREATE TABLE table1 (m text);
host1=# CREATE ROLE "user1";
host1=# ALTER ROLE "user1" WITH REPLICATION LOGIN PASSWORD  ‘user1’;
CREATE PUBLICATION host1_table1 FOR TABLE table1;
SELECT pg_create_logical_replication_slot(‘host1_table1’, ‘pgoutput’);
```
ВМ2
```sql
postgres=# CREATE DATABASE host2;
CREATE DATABASE
postgres=# \c host2
You are now connected to database "host2" as user "postgres".
host2=# CREATE TABLE table2 (m text);
host2=# CREATE ROLE "user2";
host2=# ALTER ROLE "user2" WITH REPLICATION LOGIN PASSWORD  ‘user2’;
CREATE PUBLICATION host2_table2 FOR TABLE table2;
SELECT pg_create_logical_replication_slot(‘host2_table2’, ‘pgoutput’);
```
ВМ1
```sql
host1=# CREATE TABLE table2 (m text);
CREATE SUBSCRIPTION host2_table2_sub CONNECTION 'host={ip} port=5432 user=user2 password=user2 dbname=host2' PUBLICATION host2_table2 WITH (copy_data = true);
```
ВМ2
```sql
host2=# CREATE TABLE table1 (m text);
CREATE SUBSCRIPTION host1_table1_sub CONNECTION 'host={ip} port=5432 user=user1 password=user1 dbname=host1' PUBLICATION host1_table1 WITH (copy_data = true);
```

ВМ3
```sql
```sql
host3=# CREATE TABLE table2 (m text);
host3=# CREATE TABLE table1 (m text);
CREATE SUBSCRIPTION host1_table1_sub CONNECTION 'host={ip} port=5432 user=user1 password=user1 dbname=host1' PUBLICATION host1_table1 WITH (copy_data = true);
CREATE SUBSCRIPTION host2_table2_sub CONNECTION 'host={ip} port=5432 user=user2 password=user2 dbname=host2' PUBLICATION host2_table2 WITH (copy_data = true);
```

```sql
 SHOW max_wal_senders;
 max_wal_senders 
-----------------
 10
(1 row)
```

ВМ3
```sql
su - postgres
createuser --replication -P -e replicator
echo ‘host    replication     replicator      #.#.#.#/32           md5’ >> /etc/postgresql/15/main/pg_hba.conf
psql -c "select pg_reload_conf();"
```
BM4
```sql
sudo su
systemctl stop postgresql.service
su - postgres
rm -rif /var/lib/postgresql/14/main/*
pg_basebackup -h #.#.#.# -D /var/lib/postgresql/15/main -U replicator -P -v  -R -X stream -C -S pgstandby1
exit
systemctl start postgresql.service
```
Ручное переключение на новый мастер в случае сбоя:
pg_ctlcluster 15 main promote


