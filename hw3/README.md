# Физический уровень PostgreSQL (Установка и настройка PostgreSQL)

### Подготовка стенда
Задние выполняется на виртуальной машине в VMware ESXi.
```
all:
  vars:
  ...
  
  hosts:
    test-Postgresql1:
      ansible_host: *.*.*.*
      OS_IP: '{{ ansible_host }}'
      OS_NAME: test-Postgresql1
      VM_CPU: 4
      VM_RAM: 8
      ...
```

```shell
root@test-Postgresql1:/# lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 20.04 LTS
Release:	20.04
Codename:	focal
```
установка PostgreSQL
```shell
root@test-Postgresql1:/# echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list
root@test-Postgresql1:/# wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
root@test-Postgresql1:/# apt-get update
root@test-Postgresql1:/# apt install postgresql
```
```shell
root@test-Postgresql1:/# sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

```shell
root@test-Postgresql1:/# sudo su  postgres
postgres@test-Postgresql1:/$ psql
psql (15.2 (Ubuntu 15.2-1.pgdg20.04+1))
Type "help" for help.

postgres=# CREATE DATABASE hw3;
CREATE DATABASE
postgres=# \c hw3;
You are now connected to database "hw3" as user "postgres".
hw3=# create table test(c1 text);
CREATE TABLE
hw3=# insert into test values('1');
INSERT 0 1
hw3=# \q
```
```shell
root@test-Postgresql1:/# pg_ctlcluster 15 main stop
root@test-Postgresql1:/# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
Подключение диска (без перезагрузки)
```shell
root@test-Postgresql1:/# ls /sys/class/scsi_host/ | while read host ; do echo '- - -' > /sys/class/scsi_host/$host/scan; done;s

root@test-Postgresql1:/# fdisk /dev/sdc
...
root@test-Postgresql1:/# mkfs.ext4 /dev/sdc1
...
root@test-Postgresql1:/# mkdir -p /mnt/data
root@test-Postgresql1:/# chown -R postgres:postgres /mnt/data/
root@test-Postgresql1:/# echo "/dev/disk/by-uuid/$(blkid | grep sdc | grep -E 'UUID=\"[0-9a-f]{8}(-[0-9a-f]{4}){3}-[0-9a-f]{12}\"' -o) /mnt/data ext4 defaults 0 0" >> /etc/fstab
```

```shell
root@test-Postgresql1:/# nice -n 10 rsync -rlpogDcv /var/lib/postgresql/15  /mnt/data
```
В данном случае запуск кластера будет успешным, так как была
создана копия данных. В случае же с предложенным -
ошибка (**mv – move files**).

```shell
root@test-Postgresql1:/# pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist
```
Поменяем:
```shell
/etc/postgresql/15/main/postgresql.conf:data_directory = '/var/lib/postgresql/15/main'
```
на
```shell
/etc/postgresql/15/main/postgresql.conf:data_directory = '/mnt/data/15/main'
```

```shell
root@test-Postgresql1:/# pg_ctlcluster 15 main start
root@test-Postgresql1:/# pg_ctlcluster 15 main status
pg_ctl: server is running (PID: 1454414)
/usr/lib/postgresql/15/bin/postgres "-D" "/mnt/data/15/main" "-c" "config_file=/etc/postgresql/15/main/postgresql.conf"
```
Проверка:
```shell
psql -d hw3 -c "SELECT * FROM test;"
 c1
----
 1
(1 row)
```
#### задание со звездочкой *:
_не удаляя существующий инстанс ВМ сделайте новый, поставьте
на его PostgreSQL, удалите файлы с данными из /var/lib/postgres,
перемонтируйте внешний диск который сделали ранее от первой
виртуальной машины ко второй и запустите PostgreSQL на второй
машине так чтобы он работал с данными на внешнем диске,
расскажите как вы это сделали и что в итоге получилось._

При выполнении основного задания использовался rsync - неслучайно.
В случае необходимости увеличения объема диска или переноса данных на другую vm
(когда по каким-то причинам нет возможности выполнить задачу иными способами)

1. монтирование диска
2. rsync -rlpogDcv {SourceDirectory} {TargetDirectory} (or Rsync over SSH) - без остановки кластера postgres
3. остановка кластера postgres
4. rsync {diff}
5. Монтирование диска по старому пути или изменение /etc/postgresql/15/main/postgresql.conf:data_directory (если необходимо)
6. запуск кластера 