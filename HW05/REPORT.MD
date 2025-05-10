# Домашнее задание 5

## Углубленное изучение бэкапов

### Для задания созданы ВМ

| Hostname | IP           | CPUs | MB   | Description    |
|----------|--------------|------|------|----------------|
| u8       | 192.168.0.88 | 1    | 2048 | PG master      |
| u9       | 192.168.0.89 | 1    | 2048 | PG replica     |
| u10       | 192.168.0.90 | 1    | 2048 | PG restored   |

Сразу делаем задание *, настраивая физическую репликацию с u8 на u9  

Устанавливаем postgres сервер на узлах:

```console
sudo apt install postgresql -y
```

Включем контрольные суммы и подключение по сети:

```console
systemctl stop postgresql
sudo -Eupostgres pg_checksums --pgdata /var/lib/postgresql/16/main --enable
systemctl restart postgresql

postgres=# alter system set listen_addresses ='*';
ALTER SYSTEM
sudo systemctl restart postgresql
```

### Настраиваем физическую репликацию:

__u8:__  
Создаем роль для репликации, слот и прописываем правило в hba:

```console
sudo -Eupostgres psql -t  <<SQL
create role replica LOGIN CREATEDB REPLICATION encrypted password 'secret';
SELECT pg_create_physical_replication_slot('replica_slot');
SQL

host    replication     replica             0.0.0.0/0             scram-sha-256   ->pg_hba.conf
sudo systemctl reload postgres
```

__u9:__  
Останавливаем экземпляр, очищаем PGDATA, делаем РК, используя basebackup и созданную УЗ:

```console
sudo systemctl stop postgres

sudo rm -rf $PGDATA/*
pg_basebackup -h 192.168.0.88 -W -U replica --pgdata=$PGDATA -R --slot=replica_slot
sudo chown -R postgres:postgres $PGDATA
sudo chmod 700 $PGDATA

sudo systemctl start postgresql
```

Проверяем репликацию:  
На мастере u8:  

```sql
postgres=# table pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 1543
usesysid         | 16388
usename          | replica
application_name | 16/main
client_addr      | 192.168.0.89
client_hostname  |
client_port      | 56832
backend_start    | 2025-05-10 07:07:58.211206+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/3000148
write_lsn        | 0/3000148
flush_lsn        | 0/3000148
replay_lsn       | 0/3000148
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2025-05-10 07:09:44.871493+00
```

### Настраиваем РК на реплике:  

Устанавливаем wal-g:  

```console
wget https://github.com/wal-g/wal-g/releases/download/v3.0.7/wal-g-pg-ubuntu-24.04-amd64
sudo mv ./wal-g-pg-ubuntu-24.04-amd64 /usr/bin/wal-g
sudo  chmod ugo+x /usr/bin/wal-g
```

Создаем настройки wal-g под УЗ postgres:

```console
sudo su - postgres
echo "{
    \"WALG_COMPRESSION_METHOD\": \"brotli\",
    \"WALG_DELTA_MAX_STEPS\": \"5\",
    \"WALG_FILE_PREFIX\": \"/var/lib/postgresql/walgdata\",
    \"PGDATA\": \"/var/lib/postgresql/16/main\",
    \"PGHOST\": \"/var/run/postgresql\"
}" >~/.walg.json
```

Делаем каталог для РК:

```console
sudo mkdir /var/lib/postgresql/walgdata
sudo chown postgres:postgres /var/lib/postgresql/walgdata
```

Делаем пробную РК:

```console
sudo -Eupostgres wal-g backup-push $PGDATA
sudo -Eupostgres wal-g backup-list
INFO: 2025/05/10 07:55:53.670215 List backups from storages: [default]
backup_name                   modified             wal_file_name            storage_name
base_000000010000000000000007 2025-05-10T07:55:38Z 000000010000000000000007 default
```

Настраиваем генерацию собственных WAL и архивирование на реплике u9:

```console
sudo -Eupostgres psql -t<< EOL
alter system set wal_level = replica;
alter system set archive_mode = always;
alter system set archive_command = '/usr/bin/wal-g wal-push "%p" 2>&1 | tee -a /var/log/postgresql/walg.log';
alter system set archive_timeout = 300;
alter system set restore_command = '/usr/bin/wal-g wal-fetch "%f" "%p" 2>&1 | tee -a /var/log/postgresql/walg.log';
EOL
sudo systemctl restart postgresql
```

Настраиваем pg_bench и делаем небольшой прогон для генерации WAL на мастере u8:

```console
sudo -Eupostgres createdb benchmark
sudo -Eupostgres pgbench -i -s 1 benchmark
sudo -Eupostgres pgbench --no-vacuum -c 5 -j 2 -P 30 -T 60 benchmark
```

>Сообщения в журнале wal-g подтверждают работу архивирования:
INFO: 2025/05/10 08:00:53.817251 Files will be uploaded to storage: default
INFO: 2025/05/10 08:00:54.022077 FILE PATH: 000000010000000000000009.br

### Тестирование

Тестирование восстановления СУБД под нагрузкой на u10 c реплики u9. Чтобы избежать копирования РК на восстанавливаемый сервер сделаем NFS разделяемый каталог на u9 c хранилищем РК.  

__u9:__  

```console
sudo apt-get install nfs-kernel-server -y
echo "/var/lib/postgresql/walgdata       192.168.1.90/24(rw,fsid=0,no_subtree_check,sync)" >>/etc/exports
sudo systemctl start nfs-kernel-server
sudo exportfs -a
```

__u10:__  

```console
sudo apt-get install nfs-common -y
sudo mkdir /var/lib/postgresql/walgdata
chown postgres: /var/lib/postgresql/walgdata -R
echo "192.168.0.89:/var/lib/postgresql/walgdata /var/lib/postgresql/walgdata nfs _netdev,auto  0  0" >>/etc/fstab
systemctl daemon-reload
mount -a
```

Удаляем предыдущие РК:

```console
sudo -Eupostgres wal-g delete everything --confirm
```


Запускаем pg_bench на u8 чтобы создать нагрузку:

```console
sudo -Eupostgres pgbench -c 5 -j 2 -P 30 -T 600 benchmark
pgbench (16.8 (Ubuntu 16.8-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 30.0 s, 538.3 tps, lat 9.281 ms stddev 4.923, 0 failed
...
```

Выполняем РК на u9:

```console
sudo -Eupostgres wal-g backup-push $PGDATA
```

Список РК:

```console
sudo -Eupostgres wal-g backup-list --detail
INFO: 2025/05/10 11:01:46.337803 List backups from storages: [default]
backup_name                   modified             wal_file_name            storage_name start_time           finish_time          hostname data_dir                    pg_version start_lsn  finish_lsn is_permanent
base_00000001000000000000001E 2025-05-10T11:01:38Z 00000001000000000000001E default      2025-05-10T11:01:37Z 2025-05-10T11:01:38Z u9       /var/lib/postgresql/16/main 160008     0/1E12EFB8 0/2339B978 false

sudo -Eupostgres wal-g wal-show
+-----+------------+-----------------+--------------------------+--------------------------+---------------+----------------+--------+---------------+
| TLI | PARENT TLI | SWITCHPOINT LSN | START SEGMENT            | END SEGMENT              | SEGMENT RANGE | SEGMENTS COUNT | STATUS | BACKUPS COUNT |
+-----+------------+-----------------+--------------------------+--------------------------+---------------+----------------+--------+---------------+
|   1 |          0 |             0/0 | 00000001000000000000001C | 000000010000000000000022 |             7 |              7 | OK     |             0 |
+-----+------------+-----------------+--------------------------+--------------------------+---------------+----------------+--------+---------------+
```

Восстанавливаем кластер на u10:

```console
systemctl stop postgresql
rm -rf /var/lib/postgresql/16/main/*
sudo -Eupostgres wal-g backup-fetch $PGDATA LATEST
```
>INFO: 2025/05/10 11:02:06.013191 Selecting the latest backup...
INFO: 2025/05/10 11:02:06.014406 Backup to fetch will be searched in storages: [default]
INFO: 2025/05/10 11:02:06.016185 LATEST backup is: 'base_00000001000000000000001E'
INFO: 2025/05/10 11:02:12.340335 Finished extraction of part_001.tar.br
INFO: 2025/05/10 11:02:12.357781 Finished extraction of pg_control.tar.br
INFO: 2025/05/10 11:02:12.361965 Finished extraction of backup_label.tar.br
INFO: 2025/05/10 11:02:12.361991
Backup extraction complete.

Накатим сегменты wal и запретим postgres архивировать WAL с этого сервера (хранилище то же самое в настройках, сегменты начнут перезаписываться, нарушая консистентность РК):

```console
sudo -Eupostgres touch $PGDATA/recovery.signal
chown postgres: $PGDATA -R
sed -e '/primary_/ s/^#*/#/' -i $PGDATA/postgresql.auto.conf
sed -e '/archive_/ s/^#*/#/' -i $PGDATA/postgresql.auto.conf
systemctl start postgresql
```

```sql

Получим список БД на восстановленом сервере:
postgres=# \l
                                                       List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | ICU Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+-------------+-------------+------------+-----------+-----------------------
 benchmark | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |
 postgres  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |
 template0 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
(4 rows)
```