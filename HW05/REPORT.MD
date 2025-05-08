# Домашнее задание 5

## Постоение кластера Patroni ##

Для задания созданы ВМ
| Hostname | IP           | CPUs | MB   | Description    |
|----------|--------------|------|------|----------------|
| u1       | 192.168.0.81 | 1    | 2048 | ETCD node 1    |
| u2       | 192.168.0.82 | 1    | 2048 | ETCD node 2    |
| u3       | 192.168.0.83 | 1    | 2048 | ETCD node 3    |
| u4       | 192.168.0.84 | 2    | 2048 | Patroni node 1 |
| u5       | 192.168.0.85 | 2    | 2048 | Patroni node 2 |
| u6       | 192.168.0.86 | 2    | 2048 | Patroni node 3 |
| u7       | 192.168.0.87 | 1    | 2048 | HaProxy        |

***Выполним установку etcd и первичную настройку на узлах u1-u3***

```console
sudo apt install etcd-server etcd-client -y
```

Задаим конфигурацию через переменные в /etc/default/etcd

```console
export TMP_IP_ADDR=$(hostname -i | grep -oE "([0-9]{1,3}[\.]){3}[0-9]{1,3}")
export TMP_ETCD_INITIAL_CLUSTER=u1=http://192.168.0.81:2380,u2=http://192.168.0.82:2380,u3=http://192.168.0.83:2380

sudo bash -c "echo \"ETCD_NAME=$HOSTNAME
ETCD_DATA_DIR=/var/lib/etcd/default
ETCD_LISTEN_PEER_URLS=http://$TMP_IP_ADDR:2380
ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
ETCD_ADVERTISE_CLIENT_URLS=http://$TMP_IP_ADDR:2379
ETCD_INITIAL_CLUSTER_TOKEN='etcd-cluster-1'
ETCD_INITIAL_CLUSTER=$TMP_ETCD_INITIAL_CLUSTER
ETCD_INITIAL_CLUSTER_STATE='new'
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://$TMP_IP_ADDR:2380\" > /etc/default/etcd"

sudo systemctl restart etcd
```

Проверим состояние точек подключения

```console
sudo etcdctl endpoint health  --endpoints=192.168.0.81:2379,192.168.0.82:2379,192.168.0.83:2379 -w table
+-------------------+--------+-------------+-------+
|     ENDPOINT      | HEALTH |    TOOK     | ERROR |
+-------------------+--------+-------------+-------+
| 192.168.0.81:2379 |   true |  6.009834ms |       |
| 192.168.0.83:2379 |   true |  7.982513ms |       |
| 192.168.0.82:2379 |   true | 11.359375ms |       |
+-------------------+--------+-------------+-------+

etcdctl -w table --endpoints=192.168.0.81:2379,192.168.0.82:2379,192.168.0.83:2379 endpoint status
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|     ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.0.81:2379 | f8f02439b1db49ac |  3.4.30 |   20 kB |      true |      false |        39 |          9 |                  9 |        |
| 192.168.0.82:2379 |  14ea8f772fa5197 |  3.4.30 |   20 kB |     false |      false |        39 |          9 |                  9 |        |
| 192.168.0.83:2379 | 7894cfd255567386 |  3.4.30 |   20 kB |     false |      false |        39 |          9 |                  9 |        |
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

Сделаем новым лидером произволный узел

```console
sudo etcdctl move-leader 14ea8f772fa5197 --endpoints=192.168.0.81:2379,192.168.0.82:2379,192.168.0.83:2379
Leadership transferred from f8f02439b1db49ac to 14ea8f772fa5197
sudo etcdctl -w table --endpoints=192.168.0.81:2379,192.168.0.82:2379,192.168.0.83:2379 endpoint status
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|     ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.0.81:2379 | f8f02439b1db49ac |  3.4.30 |   20 kB |     false |      false |        40 |         10 |                 10 |        |
| 192.168.0.82:2379 |  14ea8f772fa5197 |  3.4.30 |   20 kB |      true |      false |        40 |         10 |                 10 |        |
| 192.168.0.83:2379 | 7894cfd255567386 |  3.4.30 |   20 kB |     false |      false |        40 |         10 |                 10 |        |
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

***Выполним установку patroni и первичную настройку на узлах u4-u6***

```console
sudo apt install postgresql patroni -y
```

Остановим и удалим действующий экземпляр - в кластере patroni эта настройка не требуется:

```console
sudo systemctl stop postgresql
sudo systemctl disable postgresql
sudo rm -rf /var/lib/postgresql/*
```

Создадим простой конфигурационный файл patroni, настройки postgres выполняем минимально, без конфигурирования по мощности:
все узлы доступны для переезда, репликация асинронная, архивирование WAL не настраиваем пока:

```console

export TMP_IP_ADDR=$(hostname -i | grep -oE "([0-9]{1,3}[\.]){3}[0-9]{1,3}")
export TMP_ETCD_HOSTS='192.168.0.81:2379,192.168.0.82:2379,192.168.0.83:2379'

sudo bash -c "echo \"scope: PGC1
name: $HOSTNAME

restapi:
  listen: 0.0.0.0:8008
  connect_address: $TMP_IP_ADDR:8008
log:
  dir: /var/log/patroni
etcd3:
  hosts: $TMP_ETCD_HOSTS

bootstrap:  
  dcs:
    #loop_wait + 2 * retry_timeout <= ttl
    ttl: 30
    loop_wait: 10
    retry_timeout: 10    
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        listen_addresses: '*'
        port: 5432
        logging_collector: 'on'      
        archive_mode: 'off'
    slots:
      patroni:
        type: physical
  
  initdb:  
  - data-checksums
  
  pg_hba:  
  - host replication replicator 0.0.0.0/0 md5
  - host all all 0.0.0.0/0 md5

postgresql:
  listen: 0.0.0.0:5432
  connect_address: $TMP_IP_ADDR:5432
  data_dir: /pgsql/pg_data
  bin_dir: /usr/lib/postgresql/16/bin  
  authentication:
    replication:
      username: replicator
      password: secret
    superuser:
      username: postgres
      password: secret
  parameters:
    unix_socket_directories: '/tmp'
    max_connections: 300

watchdog:
  mode: automatic 
  device: /dev/watchdog
  safety_margin: 5\" >/etc/patroni/config.yml"
  ```

Создадим каталог PGDATA:

```console
sudo mkdir -p /pgsql/pg_data
sudo chown postgres:postgres /pgsql -R
sudo chmod 700 /pgsql/pg_data
```

Дадим необходимы права УЗ postgres от которой работает patroni:

```console
sudo chmod o+r /etc/patroni/config.yml
sudo mkdir /var/log/patroni
sudo chmod o+rwx /var/log/patroni
sudo modprobe softdog
sudo chown postgres:postgres /dev/watchdog
```

Зададим переменные для корректной работы patronictl в shell:

```console
sudo bash -c "echo -e \"export PATRONICTL_CONFIG_FILE=/etc/patroni/config.yml\nexport ETCDCTL_API=3\">/etc/profile.d/patroni.sh" \
&& sudo chmod +x /etc/profile.d/patroni.sh \
&& source /etc/profile.d/patroni.sh
```

Запускаем patroni и смотрим состояние кластера:

```console
patronictl list
+ Cluster: PGC1 (7501657553912294806) --------+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| u4     | 192.168.0.84 | Replica | streaming | 10 |         0 |
| u5     | 192.168.0.85 | Leader  | running   | 10 |           |
| u6     | 192.168.0.86 | Replica | running   | 10 |         0 |
+--------+--------------+---------+-----------+----+-----------+
```

Выполним пробное тестовое переключение узлов:

```console
patronictl switchover --force
Current cluster topology
+ Cluster: PGC1 (7501657553912294806) --------+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| u4     | 192.168.0.84 | Replica | streaming | 10 |         0 |
| u5     | 192.168.0.85 | Leader  | running   | 10 |           |
| u6     | 192.168.0.86 | Replica | streaming | 10 |         0 |
+--------+--------------+---------+-----------+----+-----------+
Successfully switched over to "u4"

patronictl list
+ Cluster: PGC1 (7501657553912294806) --------+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| u4     | 192.168.0.84 | Leader  | running   | 11 |           |
| u5     | 192.168.0.85 | Replica | streaming | 11 |         0 |
| u6     | 192.168.0.86 | Replica | streaming | 11 |         0 |
+--------+--------------+---------+-----------+----+-----------+
```

***Выполним установку haproxy и первичную настройку на узле u7***

```console
sudo apt install haproxy
sudo sysctl -w net.ipv4.ip_nonlocal_bind=1
sudo bash -c "echo \"net.ipv4.ip_nonlocal_bind=1\"">>/etc/sysctl.conf 
```

Создадим конфигурационный файл

```console
sudo bash -c "echo \"global
    maxconn 100

defaults
    log global    
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri \/

listen primary
    bind 192.168.0.87:5432
    option httpchk OPTIONS \/master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server u4 192.168.0.84:5432 maxconn 100 check port 8008
    server u5 192.168.0.85:5432 maxconn 100 check port 8008
    server u6 192.168.0.86:5432 maxconn 100 check port 8008

listen replica
    bind 192.168.0.87:5433
    balance leastconn
    mode tcp
    option httpchk OPTIONS \/replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server u4 192.168.0.84:5432 maxconn 100 check port 8008
    server u5 192.168.0.85:5432 maxconn 100 check port 8008
    server u6 192.168.0.86:5432 maxconn 100 check port 8008
\"" >/etc/haproxy/haproxy.cfg
```

Запустим и посмотрим что слушает haproxy

```console
sudo systemctl start haproxy
netstat -tulpn|grep haproxy
tcp        0      0 192.168.0.87:5432       0.0.0.0:*               LISTEN      15232/haproxy
tcp        0      0 192.168.0.87:5433       0.0.0.0:*               LISTEN      15232/haproxy
tcp        0      0 0.0.0.0:7000            0.0.0.0:*               LISTEN      15232/haproxy
```

Проверяем подключение  
На узле лидере создаем роль:

```sql
sudo -Eupostgres psql -c "create role test login password 'test' createdb;" 
CREATE ROLE
```

Подключаемся с win хоста

```console
PS C:\Program Files\PostgreSQL\14\scripts> .\runpsql.bat
Server [localhost]: 192.168.0.87
Database [postgres]:
Port [5432]:
Username [postgres]: test
Пароль пользователя test:
psql (14.4, сервер 16.8 (Ubuntu 16.8-0ubuntu0.24.04.1))
...
postgres=> create database test owner test;
CREATE DATABASE
```

Проверяем читающую реплику

```console
PS C:\Program Files\PostgreSQL\14\scripts> .\runpsql.bat
Server [localhost]: 192.168.0.87
Database [postgres]:
Port [5432]: 5433
Username [postgres]: test
Пароль пользователя test:
psql (14.4, сервер 16.8 (Ubuntu 16.8-0ubuntu0.24.04.1))
ПРЕДУПРЕЖДЕНИЕ: psql имеет базовую версию 14, а сервер - 16.

postgres=> create database test2 owner test;
ERROR:  cannot execute CREATE DATABASE in a read-only transaction
```

**Отработка отказа - аппаратное отключение лидирующих узлов etcd и patroni кластеров**  

*На момент проверки*  

```console
root@u1:~# etcdctl -w table --endpoints=192.168.0.81:2379,192.168.0.82:2379,192.168.0.83:2379 endpoint status
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|     ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.0.81:2379 | f8f02439b1db49ac |  3.4.30 |   94 kB |     false |      false |        40 |        305 |                305 |        |
| 192.168.0.82:2379 |  14ea8f772fa5197 |  3.4.30 |   94 kB |      true |      false |        40 |        305 |                305 |        |
| 192.168.0.83:2379 | 7894cfd255567386 |  3.4.30 |   94 kB |     false |      false |        40 |        305 |                305 |        |
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
лидирующий узел etcd - u2 c ip 192.168.0.82

+ Cluster: PGC1 (7501657553912294806) --------+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| u4     | 192.168.0.84 | Replica | streaming | 12 |         0 |
| u5     | 192.168.0.85 | Replica | streaming | 12 |         0 |
| u6     | 192.168.0.86 | Leader  | running   | 12 |           |
+--------+--------------+---------+-----------+----+-----------+
лидирующий узел patroni - u6 c ip 192.168.0.86
```

Статистика haproxy:
<html><body>
<tr class="titre"><th class="pxname" width="10%"><a name="primary"></a><a class=px href="#primary">primary</a></th><th class="empty" width="90%"></th></tr>
</table>
<table class="tbl" width="100%">
<tr class="titre"><th rowspan=2></th><th colspan=3>Queue</th><th colspan=3>Session rate</th><th colspan=6>Sessions</th><th colspan=2>Bytes</th><th colspan=2>Denied</th><th colspan=3>Errors</th><th colspan=2>Warnings</th><th colspan=9>Server</th></tr>
<tr class="titre"><th>Cur</th><th>Max</th><th>Limit</th><th>Cur</th><th>Max</th><th>Limit</th><th>Cur</th><th>Max</th><th>Limit</th><th>Total</th><th>LbTot</th><th>Last</th><th>In</th><th>Out</th><th>Req</th><th>Resp</th><th>Req</th><th>Conn</th><th>Resp</th><th>Retr</th><th>Redis</th><th>Status</th><th>LastChk</th><th>Wght</th><th>Act</th><th>Bck</th><th>Chk</th><th>Dwn</th><th>Dwntme</th><th>Thrtle</th>
</tr><tr class="frontend"><td class=ac><a name="primary/Frontend"></a><a class=lfsb href="#primary/Frontend">Frontend</a></td><td colspan=3></td><td><u>0<div class=tips><table class=det><tr><th>Current connection rate:</th><td>0/s</td></tr><tr><th>Current session rate:</th><td>0/s</td></tr></table></div></u></td><td><u>0<div class=tips><table class=det><tr><th>Max connection rate:</th><td>0/s</td></tr><tr><th>Max session rate:</th><td>0/s</td></tr></table></div></u></td><td>-</td><td>0</td><td>0</td><td>100</td><td><u>0<div class=tips><table class=det><tr><th>Cum. connections:</th><td>0</td></tr><tr><th>Cum. sessions:</th><td>0</td></tr></table></div></u></td><td></td><td></td><td>0</td><td>0<div class=tips><table class=det><tr><th>Response bytes in:</th><td>0</td></tr><tr><th>Compression in:</th><td>0</td></tr><tr><th>Compression out:</th><td>0</td><td>(0%)</td></tr><tr><th>Compression bypass:</th><td>0</td></tr><tr><th>Total bytes saved:</th><td>0</td><td>(0%)</td></tr></table></div></td><td>0</td><td>0</td><td>0</td><td></td><td></td><td></td><td></td><td class=ac>OPEN</td><td class=ac colspan=8></td></tr><tr class="active_down"><td class=ac><a name="primary/u4"></a><a class=lfsb href="#primary/u4">u4</a></td><td>0</td><td>0</td><td>-</td><td>0</td><td>0</td><td></td><td><u>0<div class=tips><table class=det><tr><th>Current active connections:</th><td>0</td></tr><tr><th>Current used connections:</th><td>0</td></tr><tr><th>Current idle connections:</th><td>0</td></tr><tr><th>- unsafe:</th><td>0</td></tr><tr><th>- safe:</th><td>0</td></tr><tr><th>Estimated need of connections:</th><td>1</td></tr><tr><th>Active connections limit:</th><td>100</td></tr><tr><th>Idle connections limit:</th><td>-</td></tr></table></div></u></td><td>0</td><td>100</td><td><u>0<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>0</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>0 / 0</td><td>ms</td></tr></table></div></u></td><td>0</td><td>?</td><td>0</td><td>0</td><td></td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>4s DOWN</td><td class=ac><u> L7STS/503 in 2ms<div class=tips>Layer7 wrong status: Service Unavailable</div></u></td><td class=ac>1/1</td><td class=ac>Y</td><td class=ac>-</td><td><u>1<div class=tips>Failed Health Checks</div></u></td><td>1</td><td>4s</td><td class=ac>-</td></tr>
<tr class="active_down"><td class=ac><a name="primary/u5"></a><a class=lfsb href="#primary/u5">u5</a></td><td>0</td><td>0</td><td>-</td><td>0</td><td>0</td><td></td><td><u>0<div class=tips><table class=det><tr><th>Current active connections:</th><td>0</td></tr><tr><th>Current used connections:</th><td>0</td></tr><tr><th>Current idle connections:</th><td>0</td></tr><tr><th>- unsafe:</th><td>0</td></tr><tr><th>- safe:</th><td>0</td></tr><tr><th>Estimated need of connections:</th><td>1</td></tr><tr><th>Active connections limit:</th><td>100</td></tr><tr><th>Idle connections limit:</th><td>-</td></tr></table></div></u></td><td>0</td><td>100</td><td><u>0<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>0</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>0 / 0</td><td>ms</td></tr></table></div></u></td><td>0</td><td>?</td><td>0</td><td>0</td><td></td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>3s DOWN</td><td class=ac><u> L7STS/503 in 1ms<div class=tips>Layer7 wrong status: Service Unavailable</div></u></td><td class=ac>1/1</td><td class=ac>Y</td><td class=ac>-</td><td><u>1<div class=tips>Failed Health Checks</div></u></td><td>1</td><td>3s</td><td class=ac>-</td></tr>
<tr class="active_up"><td class=ac><a name="primary/u6"></a><a class=lfsb href="#primary/u6">u6</a></td><td>0</td><td>0</td><td>-</td><td>0</td><td>0</td><td></td><td><u>0<div class=tips><table class=det><tr><th>Current active connections:</th><td>0</td></tr><tr><th>Current used connections:</th><td>0</td></tr><tr><th>Current idle connections:</th><td>0</td></tr><tr><th>- unsafe:</th><td>0</td></tr><tr><th>- safe:</th><td>0</td></tr><tr><th>Estimated need of connections:</th><td>1</td></tr><tr><th>Active connections limit:</th><td>100</td></tr><tr><th>Idle connections limit:</th><td>-</td></tr></table></div></u></td><td>0</td><td>100</td><td><u>0<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>0</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>0 / 0</td><td>ms</td></tr></table></div></u></td><td>0</td><td>?</td><td>0</td><td>0</td><td></td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>4s UP</td><td class=ac><u> L7OK/200 in 2ms<div class=tips>Layer7 check passed</div></u></td><td class=ac>1/1</td><td class=ac>Y</td><td class=ac>-</td><td><u>0<div class=tips>Failed Health Checks</div></u></td><td>0</td><td>0s</td><td class=ac>-</td></tr>
<tr class="backend"><td class=ac><a name="primary/Backend"></a><a class=lfsb href="#primary/Backend">Backend</a></td><td>0</td><td>0</td><td></td><td>0</td><td>0</td><td></td><td>0</td><td>0</td><td>10</td><td><u>0<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>0</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>0 / 0</td><td>ms</td></tr></table></div></u></td><td>0</td><td>?</td><td>0</td><td>0<div class=tips><table class=det><tr><th>Response bytes in:</th><td>0</td></tr><tr><th>Compression in:</th><td>0</td></tr><tr><th>Compression out:</th><td>0</td><td>(0%)</td></tr><tr><th>Compression bypass:</th><td>0</td></tr><tr><th>Total bytes saved:</th><td>0</td><td>(0%)</td></tr></table></div></td><td>0</td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>4s UP</td><td class=ac>&nbsp;</td><td class=ac>1/1</td><td class=ac>1</td><td class=ac>0</td><td class=ac>&nbsp;</td><td>0</td><td>0s</td><td></td></tr></table><p>
<table class="tbl" width="100%">
<tr class="titre"><th class="pxname" width="10%"><a name="replica"></a><a class=px href="#replica">replica</a></th><th class="empty" width="90%"></th></tr>
</table>
<table class="tbl" width="100%">
<tr class="titre"><th rowspan=2></th><th colspan=3>Queue</th><th colspan=3>Session rate</th><th colspan=6>Sessions</th><th colspan=2>Bytes</th><th colspan=2>Denied</th><th colspan=3>Errors</th><th colspan=2>Warnings</th><th colspan=9>Server</th></tr>
<tr class="titre"><th>Cur</th><th>Max</th><th>Limit</th><th>Cur</th><th>Max</th><th>Limit</th><th>Cur</th><th>Max</th><th>Limit</th><th>Total</th><th>LbTot</th><th>Last</th><th>In</th><th>Out</th><th>Req</th><th>Resp</th><th>Req</th><th>Conn</th><th>Resp</th><th>Retr</th><th>Redis</th><th>Status</th><th>LastChk</th><th>Wght</th><th>Act</th><th>Bck</th><th>Chk</th><th>Dwn</th><th>Dwntme</th><th>Thrtle</th>
</tr><tr class="frontend"><td class=ac><a name="replica/Frontend"></a><a class=lfsb href="#replica/Frontend">Frontend</a></td><td colspan=3></td><td><u>0<div class=tips><table class=det><tr><th>Current connection rate:</th><td>0/s</td></tr><tr><th>Current session rate:</th><td>0/s</td></tr></table></div></u></td><td><u>0<div class=tips><table class=det><tr><th>Max connection rate:</th><td>0/s</td></tr><tr><th>Max session rate:</th><td>0/s</td></tr></table></div></u></td><td>-</td><td>0</td><td>0</td><td>100</td><td><u>0<div class=tips><table class=det><tr><th>Cum. connections:</th><td>0</td></tr><tr><th>Cum. sessions:</th><td>0</td></tr></table></div></u></td><td></td><td></td><td>0</td><td>0<div class=tips><table class=det><tr><th>Response bytes in:</th><td>0</td></tr><tr><th>Compression in:</th><td>0</td></tr><tr><th>Compression out:</th><td>0</td><td>(0%)</td></tr><tr><th>Compression bypass:</th><td>0</td></tr><tr><th>Total bytes saved:</th><td>0</td><td>(0%)</td></tr></table></div></td><td>0</td><td>0</td><td>0</td><td></td><td></td><td></td><td></td><td class=ac>OPEN</td><td class=ac colspan=8></td></tr><tr class="active_up"><td class=ac><a name="replica/u4"></a><a class=lfsb href="#replica/u4">u4</a></td><td>0</td><td>0</td><td>-</td><td>0</td><td>0</td><td></td><td><u>0<div class=tips><table class=det><tr><th>Current active connections:</th><td>0</td></tr><tr><th>Current used connections:</th><td>0</td></tr><tr><th>Current idle connections:</th><td>0</td></tr><tr><th>- unsafe:</th><td>0</td></tr><tr><th>- safe:</th><td>0</td></tr><tr><th>Estimated need of connections:</th><td>1</td></tr><tr><th>Active connections limit:</th><td>100</td></tr><tr><th>Idle connections limit:</th><td>-</td></tr></table></div></u></td><td>0</td><td>100</td><td><u>0<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>0</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>0 / 0</td><td>ms</td></tr></table></div></u></td><td>0</td><td>?</td><td>0</td><td>0</td><td></td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>4s UP</td><td class=ac><u> L7OK/200 in 2ms<div class=tips>Layer7 check passed</div></u></td><td class=ac>1/1</td><td class=ac>Y</td><td class=ac>-</td><td><u>0<div class=tips>Failed Health Checks</div></u></td><td>0</td><td>0s</td><td class=ac>-</td></tr>
<tr class="active_up"><td class=ac><a name="replica/u5"></a><a class=lfsb href="#replica/u5">u5</a></td><td>0</td><td>0</td><td>-</td><td>0</td><td>0</td><td></td><td><u>0<div class=tips><table class=det><tr><th>Current active connections:</th><td>0</td></tr><tr><th>Current used connections:</th><td>0</td></tr><tr><th>Current idle connections:</th><td>0</td></tr><tr><th>- unsafe:</th><td>0</td></tr><tr><th>- safe:</th><td>0</td></tr><tr><th>Estimated need of connections:</th><td>1</td></tr><tr><th>Active connections limit:</th><td>100</td></tr><tr><th>Idle connections limit:</th><td>-</td></tr></table></div></u></td><td>0</td><td>100</td><td><u>0<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>0</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>0 / 0</td><td>ms</td></tr></table></div></u></td><td>0</td><td>?</td><td>0</td><td>0</td><td></td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>4s UP</td><td class=ac><u> L7OK/200 in 2ms<div class=tips>Layer7 check passed</div></u></td><td class=ac>1/1</td><td class=ac>Y</td><td class=ac>-</td><td><u>0<div class=tips>Failed Health Checks</div></u></td><td>0</td><td>0s</td><td class=ac>-</td></tr>
<tr class="active_down"><td class=ac><a name="replica/u6"></a><a class=lfsb href="#replica/u6">u6</a></td><td>0</td><td>0</td><td>-</td><td>0</td><td>0</td><td></td><td><u>0<div class=tips><table class=det><tr><th>Current active connections:</th><td>0</td></tr><tr><th>Current used connections:</th><td>0</td></tr><tr><th>Current idle connections:</th><td>0</td></tr><tr><th>- unsafe:</th><td>0</td></tr><tr><th>- safe:</th><td>0</td></tr><tr><th>Estimated need of connections:</th><td>1</td></tr><tr><th>Active connections limit:</th><td>100</td></tr><tr><th>Idle connections limit:</th><td>-</td></tr></table></div></u></td><td>0</td><td>100</td><td><u>0<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>0</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>0 / 0</td><td>ms</td></tr></table></div></u></td><td>0</td><td>?</td><td>0</td><td>0</td><td></td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>1s DOWN</td><td class=ac><u> L7STS/503 in 2ms<div class=tips>Layer7 wrong status: Service Unavailable</div></u></td><td class=ac>1/1</td><td class=ac>Y</td><td class=ac>-</td><td><u>1<div class=tips>Failed Health Checks</div></u></td><td>1</td><td>1s</td><td class=ac>-</td></tr>
<tr class="backend"><td class=ac><a name="replica/Backend"></a><a class=lfsb href="#replica/Backend">Backend</a></td><td>0</td><td>0</td><td></td><td>0</td><td>0</td><td></td><td>0</td><td>0</td><td>10</td><td><u>0<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>0</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>0 / 0</td><td>ms</td></tr></table></div></u></td><td>0</td><td>?</td><td>0</td><td>0<div class=tips><table class=det><tr><th>Response bytes in:</th><td>0</td></tr><tr><th>Compression in:</th><td>0</td></tr><tr><th>Compression out:</th><td>0</td><td>(0%)</td></tr><tr><th>Compression bypass:</th><td>0</td></tr><tr><th>Total bytes saved:</th><td>0</td><td>(0%)</td></tr></table></div></td><td>0</td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>4s UP</td><td class=ac>&nbsp;</td><td class=ac>2/2</td><td class=ac>2</td><td class=ac>0</td><td class=ac>&nbsp;</td><td>0</td><td>0s</td><td></td></tr></table><p>
</body></html>

Отключаем питание ВМ u2,u6  

*Состяние etcd после отключения*  

Новые лидеры - u3 и u5:
```console
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|     ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.0.81:2379 | f8f02439b1db49ac |  3.4.30 |   94 kB |     false |      false |        42 |        310 |                310 |        |
| 192.168.0.83:2379 | 7894cfd255567386 |  3.4.30 |   94 kB |      true |      false |        42 |        310 |                310 |        |
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
+ Cluster: PGC1 (7501657553912294806) --------+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| u4     | 192.168.0.84 | Replica | streaming | 13 |         0 |
| u5     | 192.168.0.85 | Leader  | running   | 13 |           |
+--------+--------------+---------+-----------+----+-----------+
```

Состояние haproxy:

<html><body>
<table class="tbl" width="100%">
<tr class="titre"><th class="pxname" width="10%"><a name="stats"></a><a class=px href="#stats">stats</a></th><th class="empty" width="90%"></th></tr>
</table>
<table class="tbl" width="100%">
<tr class="titre"><th rowspan=2></th><th colspan=3>Queue</th><th colspan=3>Session rate</th><th colspan=6>Sessions</th><th colspan=2>Bytes</th><th colspan=2>Denied</th><th colspan=3>Errors</th><th colspan=2>Warnings</th><th colspan=9>Server</th></tr>
<tr class="titre"><th>Cur</th><th>Max</th><th>Limit</th><th>Cur</th><th>Max</th><th>Limit</th><th>Cur</th><th>Max</th><th>Limit</th><th>Total</th><th>LbTot</th><th>Last</th><th>In</th><th>Out</th><th>Req</th><th>Resp</th><th>Req</th><th>Conn</th><th>Resp</th><th>Retr</th><th>Redis</th><th>Status</th><th>LastChk</th><th>Wght</th><th>Act</th><th>Bck</th><th>Chk</th><th>Dwn</th><th>Dwntme</th><th>Thrtle</th>
</tr><tr class="frontend"><td class=ac><a name="stats/Frontend"></a><a class=lfsb href="#stats/Frontend">Frontend</a></td><td colspan=3></td><td><u>2<div class=tips><table class=det><tr><th>Current connection rate:</th><td>2/s</td></tr><tr><th>Current session rate:</th><td>2/s</td></tr><tr><th>Current request rate:</th><td>1/s</td></tr></table></div></u></td><td><u>2<div class=tips><table class=det><tr><th>Max connection rate:</th><td>2/s</td></tr><tr><th>Max session rate:</th><td>2/s</td></tr><tr><th>Max request rate:</th><td>1/s</td></tr></table></div></u></td><td>-</td><td>2</td><td>2</td><td>100</td><td><u>4<div class=tips><table class=det><tr><th>Cum. connections:</th><td>4</td></tr><tr><th>Cum. sessions:</th><td>4</td></tr><tr><th>- HTTP/1 sessions:</th><td>4</td></tr><tr><th>- HTTP/2 sessions:</th><td>0</td></tr><tr><th>- HTTP/3 sessions:</th><td>0</td></tr><tr><th>- other sessions:</th><td>0</td></tr><tr><th>Cum. HTTP requests:</th><td>4</td></tr><tr><th>- HTTP/1 requests:</th><td>4</td></tr><tr><th>- HTTP/2 requests:</th><td>0</td></tr><tr><th>- HTTP/3 requests:</th><td>0</td></tr><tr><th>- other requests:</th><td>0</td></tr><tr><th>- HTTP 1xx responses:</th><td>0</td></tr><tr><th>- HTTP 2xx responses:</th><td>2</td></tr><tr><th>&nbsp;&nbsp;Compressed 2xx:</th><td>0</td><td>(0%)</td></tr><tr><th>- HTTP 3xx responses:</th><td>0</td></tr><tr><th>- HTTP 4xx responses:</th><td>1</td></tr><tr><th>- HTTP 5xx responses:</th><td>0</td></tr><tr><th>- other responses:</th><td>0</td></tr><tr><th>Intercepted requests:</th><td>3</td></tr><tr><th>Cache lookups:</th><td>0</td></tr><tr><th>Cache hits:</th><td>0</td><td>(0%)</td></tr><tr><th>Failed hdr rewrites:</th><td>0</td></tr><tr><th>Internal errors:</th><td>0</td></tr></table></div></u></td><td></td><td></td><td>820</td><td>5<span class="rls">7</span>830<div class=tips><table class=det><tr><th>Response bytes in:</th><td>5<span class="rls">7</span>830</td></tr><tr><th>Compression in:</th><td>0</td></tr><tr><th>Compression out:</th><td>0</td><td>(0%)</td></tr><tr><th>Compression bypass:</th><td>0</td></tr><tr><th>Total bytes saved:</th><td>0</td><td>(0%)</td></tr></table></div></td><td>0</td><td>0</td><td>1</td><td></td><td></td><td></td><td></td><td class=ac>OPEN</td><td class=ac colspan=8></td></tr><tr class="backend"><td class=ac><a name="stats/Backend"></a><a class=lfsb href="#stats/Backend">Backend</a></td><td>0</td><td>0</td><td></td><td>0</td><td>0</td><td></td><td>0</td><td>0</td><td>10</td><td><u>0<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>0</td></tr><tr><th>New connections:</th><td>0</td></tr><tr><th>Reused connections:</th><td>0</td><td>(0%)</td></tr><tr><th>Cum. HTTP requests:</th><td>0</td></tr><tr><th>- HTTP 1xx responses:</th><td>0</td></tr><tr><th>- HTTP 2xx responses:</th><td>0</td></tr><tr><th>&nbsp;&nbsp;Compressed 2xx:</th><td>0</td><td>(0%)</td></tr><tr><th>- HTTP 3xx responses:</th><td>0</td></tr><tr><th>- HTTP 4xx responses:</th><td>0</td></tr><tr><th>- HTTP 5xx responses:</th><td>0</td></tr><tr><th>- other responses:</th><td>0</td></tr><tr><th>Cache lookups:</th><td>0</td></tr><tr><th>Cache hits:</th><td>0</td><td>(0%)</td></tr><tr><th>Failed hdr rewrites:</th><td>0</td></tr><tr><th>Internal errors:</th><td>0</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Responses time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>3<span class="rls">0</span>217 / 60</td><td>ms</td></tr></table></div></u></td><td>0</td><td>0s</td><td>820</td><td>5<span class="rls">7</span>830<div class=tips><table class=det><tr><th>Response bytes in:</th><td>5<span class="rls">7</span>830</td></tr><tr><th>Compression in:</th><td>0</td></tr><tr><th>Compression out:</th><td>0</td><td>(0%)</td></tr><tr><th>Compression bypass:</th><td>0</td></tr><tr><th>Total bytes saved:</th><td>0</td><td>(0%)</td></tr></table></div></td><td>0</td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>21m30s UP</td><td class=ac>&nbsp;</td><td class=ac>0/0</td><td class=ac>0</td><td class=ac>0</td><td class=ac>&nbsp;</td><td>0</td><td>&nbsp;</td><td></td></tr></table><p>
<table class="tbl" width="100%">
<tr class="titre"><th class="pxname" width="10%"><a name="primary"></a><a class=px href="#primary">primary</a></th><th class="empty" width="90%"></th></tr>
</table>
<table class="tbl" width="100%">
<tr class="titre"><th rowspan=2></th><th colspan=3>Queue</th><th colspan=3>Session rate</th><th colspan=6>Sessions</th><th colspan=2>Bytes</th><th colspan=2>Denied</th><th colspan=3>Errors</th><th colspan=2>Warnings</th><th colspan=9>Server</th></tr>
<tr class="titre"><th>Cur</th><th>Max</th><th>Limit</th><th>Cur</th><th>Max</th><th>Limit</th><th>Cur</th><th>Max</th><th>Limit</th><th>Total</th><th>LbTot</th><th>Last</th><th>In</th><th>Out</th><th>Req</th><th>Resp</th><th>Req</th><th>Conn</th><th>Resp</th><th>Retr</th><th>Redis</th><th>Status</th><th>LastChk</th><th>Wght</th><th>Act</th><th>Bck</th><th>Chk</th><th>Dwn</th><th>Dwntme</th><th>Thrtle</th>
</tr><tr class="frontend"><td class=ac><a name="primary/Frontend"></a><a class=lfsb href="#primary/Frontend">Frontend</a></td><td colspan=3></td><td><u>0<div class=tips><table class=det><tr><th>Current connection rate:</th><td>0/s</td></tr><tr><th>Current session rate:</th><td>0/s</td></tr></table></div></u></td><td><u>0<div class=tips><table class=det><tr><th>Max connection rate:</th><td>0/s</td></tr><tr><th>Max session rate:</th><td>0/s</td></tr></table></div></u></td><td>-</td><td>0</td><td>0</td><td>100</td><td><u>0<div class=tips><table class=det><tr><th>Cum. connections:</th><td>0</td></tr><tr><th>Cum. sessions:</th><td>0</td></tr></table></div></u></td><td></td><td></td><td>0</td><td>0<div class=tips><table class=det><tr><th>Response bytes in:</th><td>0</td></tr><tr><th>Compression in:</th><td>0</td></tr><tr><th>Compression out:</th><td>0</td><td>(0%)</td></tr><tr><th>Compression bypass:</th><td>0</td></tr><tr><th>Total bytes saved:</th><td>0</td><td>(0%)</td></tr></table></div></td><td>0</td><td>0</td><td>0</td><td></td><td></td><td></td><td></td><td class=ac>OPEN</td><td class=ac colspan=8></td></tr><tr class="active_down"><td class=ac><a name="primary/u4"></a><a class=lfsb href="#primary/u4">u4</a></td><td>0</td><td>0</td><td>-</td><td>0</td><td>0</td><td></td><td><u>0<div class=tips><table class=det><tr><th>Current active connections:</th><td>0</td></tr><tr><th>Current used connections:</th><td>0</td></tr><tr><th>Current idle connections:</th><td>0</td></tr><tr><th>- unsafe:</th><td>0</td></tr><tr><th>- safe:</th><td>0</td></tr><tr><th>Estimated need of connections:</th><td>1</td></tr><tr><th>Active connections limit:</th><td>100</td></tr><tr><th>Idle connections limit:</th><td>-</td></tr></table></div></u></td><td>0</td><td>100</td><td><u>0<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>0</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>0 / 0</td><td>ms</td></tr></table></div></u></td><td>0</td><td>?</td><td>0</td><td>0</td><td></td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>21m30s DOWN</td><td class=ac><u> L7STS/503 in 2ms<div class=tips>Layer7 wrong status: Service Unavailable</div></u></td><td class=ac>1/1</td><td class=ac>Y</td><td class=ac>-</td><td><u>1<div class=tips>Failed Health Checks</div></u></td><td>1</td><td>21m30s</td><td class=ac>-</td></tr>
<tr class="active_up"><td class=ac><a name="primary/u5"></a><a class=lfsb href="#primary/u5">u5</a></td><td>0</td><td>0</td><td>-</td><td>0</td><td>0</td><td></td><td><u>0<div class=tips><table class=det><tr><th>Current active connections:</th><td>0</td></tr><tr><th>Current used connections:</th><td>0</td></tr><tr><th>Current idle connections:</th><td>0</td></tr><tr><th>- unsafe:</th><td>0</td></tr><tr><th>- safe:</th><td>0</td></tr><tr><th>Estimated need of connections:</th><td>1</td></tr><tr><th>Active connections limit:</th><td>100</td></tr><tr><th>Idle connections limit:</th><td>-</td></tr></table></div></u></td><td>0</td><td>100</td><td><u>0<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>0</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>0 / 0</td><td>ms</td></tr></table></div></u></td><td>0</td><td>?</td><td>0</td><td>0</td><td></td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>2m40s UP</td><td class=ac><u> L7OK/200 in 2ms<div class=tips>Layer7 check passed</div></u></td><td class=ac>1/1</td><td class=ac>Y</td><td class=ac>-</td><td><u>1<div class=tips>Failed Health Checks</div></u></td><td>1</td><td>18m49s</td><td class=ac>-</td></tr>
<tr class="active_down"><td class=ac><a name="primary/u6"></a><a class=lfsb href="#primary/u6">u6</a></td><td>0</td><td>0</td><td>-</td><td>0</td><td>0</td><td></td><td><u>0<div class=tips><table class=det><tr><th>Current active connections:</th><td>0</td></tr><tr><th>Current used connections:</th><td>0</td></tr><tr><th>Current idle connections:</th><td>0</td></tr><tr><th>- unsafe:</th><td>0</td></tr><tr><th>- safe:</th><td>0</td></tr><tr><th>Estimated need of connections:</th><td>1</td></tr><tr><th>Active connections limit:</th><td>100</td></tr><tr><th>Idle connections limit:</th><td>-</td></tr></table></div></u></td><td>0</td><td>100</td><td><u>0<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>0</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>0 / 0</td><td>ms</td></tr></table></div></u></td><td>0</td><td>?</td><td>0</td><td>0</td><td></td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>3m3s DOWN</td><td class=ac><u> * L4TOUT in 3002ms<div class=tips>Layer4 timeout</div></u></td><td class=ac>1/1</td><td class=ac>Y</td><td class=ac>-</td><td><u>3<div class=tips>Failed Health Checks</div></u></td><td>1</td><td>3m3s</td><td class=ac>-</td></tr>
<tr class="backend"><td class=ac><a name="primary/Backend"></a><a class=lfsb href="#primary/Backend">Backend</a></td><td>0</td><td>0</td><td></td><td>0</td><td>0</td><td></td><td>0</td><td>0</td><td>10</td><td><u>0<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>0</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>0 / 0</td><td>ms</td></tr></table></div></u></td><td>0</td><td>?</td><td>0</td><td>0<div class=tips><table class=det><tr><th>Response bytes in:</th><td>0</td></tr><tr><th>Compression in:</th><td>0</td></tr><tr><th>Compression out:</th><td>0</td><td>(0%)</td></tr><tr><th>Compression bypass:</th><td>0</td></tr><tr><th>Total bytes saved:</th><td>0</td><td>(0%)</td></tr></table></div></td><td>0</td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>2m40s UP</td><td class=ac>&nbsp;</td><td class=ac>1/1</td><td class=ac>1</td><td class=ac>0</td><td class=ac>&nbsp;</td><td>1</td><td>23s</td><td></td></tr></table><p>
<table class="tbl" width="100%">
<tr class="titre"><th class="pxname" width="10%"><a name="replica"></a><a class=px href="#replica">replica</a></th><th class="empty" width="90%"></th></tr>
</table>
<table class="tbl" width="100%">
<tr class="titre"><th rowspan=2></th><th colspan=3>Queue</th><th colspan=3>Session rate</th><th colspan=6>Sessions</th><th colspan=2>Bytes</th><th colspan=2>Denied</th><th colspan=3>Errors</th><th colspan=2>Warnings</th><th colspan=9>Server</th></tr>
<tr class="titre"><th>Cur</th><th>Max</th><th>Limit</th><th>Cur</th><th>Max</th><th>Limit</th><th>Cur</th><th>Max</th><th>Limit</th><th>Total</th><th>LbTot</th><th>Last</th><th>In</th><th>Out</th><th>Req</th><th>Resp</th><th>Req</th><th>Conn</th><th>Resp</th><th>Retr</th><th>Redis</th><th>Status</th><th>LastChk</th><th>Wght</th><th>Act</th><th>Bck</th><th>Chk</th><th>Dwn</th><th>Dwntme</th><th>Thrtle</th>
</tr><tr class="frontend"><td class=ac><a name="replica/Frontend"></a><a class=lfsb href="#replica/Frontend">Frontend</a></td><td colspan=3></td><td><u>0<div class=tips><table class=det><tr><th>Current connection rate:</th><td>0/s</td></tr><tr><th>Current session rate:</th><td>0/s</td></tr></table></div></u></td><td><u>1<div class=tips><table class=det><tr><th>Max connection rate:</th><td>1/s</td></tr><tr><th>Max session rate:</th><td>1/s</td></tr></table></div></u></td><td>-</td><td>0</td><td>1</td><td>100</td><td><u>4<div class=tips><table class=det><tr><th>Cum. connections:</th><td>4</td></tr><tr><th>Cum. sessions:</th><td>4</td></tr></table></div></u></td><td></td><td></td><td>744</td><td><span class="rls">1</span>025<div class=tips><table class=det><tr><th>Response bytes in:</th><td><span class="rls">1</span>025</td></tr><tr><th>Compression in:</th><td>0</td></tr><tr><th>Compression out:</th><td>0</td><td>(0%)</td></tr><tr><th>Compression bypass:</th><td>0</td></tr><tr><th>Total bytes saved:</th><td>0</td><td>(0%)</td></tr></table></div></td><td>0</td><td>0</td><td>0</td><td></td><td></td><td></td><td></td><td class=ac>OPEN</td><td class=ac colspan=8></td></tr><tr class="active_up"><td class=ac><a name="replica/u4"></a><a class=lfsb href="#replica/u4">u4</a></td><td>0</td><td>0</td><td>-</td><td>0</td><td>1</td><td></td><td><u>0<div class=tips><table class=det><tr><th>Current active connections:</th><td>0</td></tr><tr><th>Current used connections:</th><td>0</td></tr><tr><th>Current idle connections:</th><td>0</td></tr><tr><th>- unsafe:</th><td>0</td></tr><tr><th>- safe:</th><td>0</td></tr><tr><th>Estimated need of connections:</th><td>1</td></tr><tr><th>Active connections limit:</th><td>100</td></tr><tr><th>Idle connections limit:</th><td>-</td></tr></table></div></u></td><td>1</td><td>100</td><td><u>2<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>2</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>6 / 6</td><td>ms</td></tr></table></div></u></td><td>2</td><td>6m41s</td><td>186</td><td>50</td><td></td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>21m30s UP</td><td class=ac><u> L7OK/200 in 2ms<div class=tips>Layer7 check passed</div></u></td><td class=ac>1/1</td><td class=ac>Y</td><td class=ac>-</td><td><u>0<div class=tips>Failed Health Checks</div></u></td><td>0</td><td>0s</td><td class=ac>-</td></tr>
<tr class="active_down"><td class=ac><a name="replica/u5"></a><a class=lfsb href="#replica/u5">u5</a></td><td>0</td><td>0</td><td>-</td><td>0</td><td>1</td><td></td><td><u>0<div class=tips><table class=det><tr><th>Current active connections:</th><td>0</td></tr><tr><th>Current used connections:</th><td>0</td></tr><tr><th>Current idle connections:</th><td>0</td></tr><tr><th>- unsafe:</th><td>0</td></tr><tr><th>- safe:</th><td>0</td></tr><tr><th>Estimated need of connections:</th><td>1</td></tr><tr><th>Active connections limit:</th><td>100</td></tr><tr><th>Idle connections limit:</th><td>-</td></tr></table></div></u></td><td>1</td><td>100</td><td><u>2<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>2</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>5<span class="rls">6</span>640 / 2<span class="rls">8</span>333</td><td>ms</td></tr></table></div></u></td><td>2</td><td>6m38s</td><td>558</td><td>975</td><td></td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>2m35s DOWN</td><td class=ac><u> L7STS/503 in 2ms<div class=tips>Layer7 wrong status: Service Unavailable</div></u></td><td class=ac>1/1</td><td class=ac>Y</td><td class=ac>-</td><td><u>3<div class=tips>Failed Health Checks</div></u></td><td>1</td><td>2m35s</td><td class=ac>-</td></tr>
<tr class="active_down"><td class=ac><a name="replica/u6"></a><a class=lfsb href="#replica/u6">u6</a></td><td>0</td><td>0</td><td>-</td><td>0</td><td>0</td><td></td><td><u>0<div class=tips><table class=det><tr><th>Current active connections:</th><td>0</td></tr><tr><th>Current used connections:</th><td>0</td></tr><tr><th>Current idle connections:</th><td>0</td></tr><tr><th>- unsafe:</th><td>0</td></tr><tr><th>- safe:</th><td>0</td></tr><tr><th>Estimated need of connections:</th><td>1</td></tr><tr><th>Active connections limit:</th><td>100</td></tr><tr><th>Idle connections limit:</th><td>-</td></tr></table></div></u></td><td>0</td><td>100</td><td><u>0<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>0</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>0 / 0</td><td>ms</td></tr></table></div></u></td><td>0</td><td>?</td><td>0</td><td>0</td><td></td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>21m27s DOWN</td><td class=ac><u> * L4TOUT in 3001ms<div class=tips>Layer4 timeout</div></u></td><td class=ac>1/1</td><td class=ac>Y</td><td class=ac>-</td><td><u>1<div class=tips>Failed Health Checks</div></u></td><td>1</td><td>21m27s</td><td class=ac>-</td></tr>
<tr class="backend"><td class=ac><a name="replica/Backend"></a><a class=lfsb href="#replica/Backend">Backend</a></td><td>0</td><td>0</td><td></td><td>0</td><td>1</td><td></td><td>0</td><td>1</td><td>10</td><td><u>4<div class=tips><table class=det><tr><th>Cum. sessions:</th><td>4</td></tr><tr><th colspan=3>Max / Avg over last 1024 success. conn.</th></tr><tr><th>- Queue time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Connect time:</th><td>0 / 0</td><td>ms</td></tr><tr><th>- Total time:</th><td>5<span class="rls">6</span>640 / 1<span class="rls">4</span>169</td><td>ms</td></tr></table></div></u></td><td>4</td><td>6m38s</td><td>744</td><td><span class="rls">1</span>025<div class=tips><table class=det><tr><th>Response bytes in:</th><td><span class="rls">1</span>025</td></tr><tr><th>Compression in:</th><td>0</td></tr><tr><th>Compression out:</th><td>0</td><td>(0%)</td></tr><tr><th>Compression bypass:</th><td>0</td></tr><tr><th>Total bytes saved:</th><td>0</td><td>(0%)</td></tr></table></div></td><td>0</td><td>0</td><td></td><td>0</td><td><u>0<div class=tips>Connection resets during transfers: 0 client, 0 server</div></u></td><td>0</td><td>0</td><td class=ac>21m30s UP</td><td class=ac>&nbsp;</td><td class=ac>1/1</td><td class=ac>1</td><td class=ac>0</td><td class=ac>&nbsp;</td><td>0</td><td>0s</td><td></td></tr></table><p>
</body></html>


## Задание * ##
Выполняем настройку РК с помощью pg_probackup.  
Сравнив варианты возможной организации РК на этом кластере и доступные в публичном версии pg_probackup я выбрал вариант с резервированием одного из узлов кластера под РК и выполнения локальной РК СУБД и WAL.  
Варианты с поддержкой pg_probackup на S3 требует доступа в закрытую часть репозитория postgrespro, а использование отдельного сервера РК и подключением через балансировщик слишком громоздко для задания, хоть и реализуемо.  

Отмаркируем узел u6 как nofailover:

```console
echo "tags:
    nofailover: true" >>/etc/patroni/config.yml
```

и включим генерацию журналов на нем:

```sql
alter system set archive_mode=always;
```
```console
systemctl restart patroni
patronictl list
+ Cluster: PGC1 (7501657553912294806) --------+----+-----------+------------------+
| Member | Host         | Role    | State     | TL | Lag in MB | Tags             |
+--------+--------------+---------+-----------+----+-----------+------------------+
| u6     | 192.168.0.86 | Replica | streaming | 15 |         0 | nofailover: true |
+--------+--------------+---------+-----------+----+-----------+------------------+
```
```sql
postgres=# show archive_mode ;
 archive_mode
--------------
 always
```

Инициализируем инстанс РК
```console
sudo mkdir -p /pgsql/pg_backup
sudo chown -R postgres:postgres /pgsql/pg_backup
sudo -Eupostgres pg_probackup init -B /pgsql/pg_backup
sudo -Eupostgres sudo pg_probackup add-instance -B /pgsql/pg_backup -D /pgsql/pg_data --instance PGPBACKUP1
sudo -Eupostgres sudo pg_probackup set-config -B /pgsql/pg_backup --instance PGPBACKUP1  --log-directory=/pgsql/pg_data/log --log-rotation-size=10MB --log-level-file=info
```

Зададим политику хранения - одна полная РК и всё, что к ней относится:
```console
pg_probackup set-config -B /pgsql/pg_backup --instance PGPBACKUP1 --retention-redundancy=1
```

Создадим УЗ для РК:
```console
sudo -Eu postgres psql -t  <<SQL
BEGIN;
CREATE ROLE backup WITH LOGIN REPLICATION ENCRYPTED PASSWORD 'secret';
GRANT USAGE ON SCHEMA pg_catalog TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.current_setting(text) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_is_in_recovery() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_start(text, boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_stop(boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_create_restore_point(text) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_switch_wal() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_last_wal_replay_lsn() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current_snapshot() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_snapshot_xmax(txid_snapshot) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_control_checkpoint() TO backup;
COMMIT;
SQL
```
Сохраним пароль для беспарольного выполнения РК
```console
printf "*:*:*:backup:secret">>$(getent passwd "postgres" | cut -d: -f6)/.pgpass
chown postgres:postgres $(getent passwd "postgres" | cut -d: -f6)/.pgpass
chmod 600 $(getent passwd "postgres" | cut -d: -f6)/.pgpass
```

Настроим РК WAL для PITR:
```console
sudo -Eu postgres psql -t  <<SQL
ALTER SYSTEM SET archive_command='/usr/bin/pg_probackup archive-push -B /pgsql/pg_backup --instance PGPBACKUP1 -U backup -w --wal-file-path=%p --wal-file-name=%f --overwrite --compress';
ALTER SYSTEM SET archive_mode TO 'on';
ALTER SYSTEM SET wal_keep_size TO 2048;
ALTER SYSTEM SET max_wal_senders=10;
ALTER SYSTEM SET archive_timeout=240;
SQL
systemctl restart patroni
```

Выполним полное РК:
```console
su - postgres -c 'pg_probackup backup -U backup -w -d postgres -B /pgsql/pg_backup --instance PGPBACKUP1 -b FULL --stream --temp-slot --compress';
su - postgres -c 'pg_probackup show -B /pgsql/pg_backup --instance PGPBACKUP1'
====================================================================================================================================
 Instance    Version  ID      Recovery Time           Mode  WAL Mode  TLI   Time   Data   WAL  Zratio  Start LSN  Stop LSN   Status
====================================================================================================================================
 PGPBACKUP1  16       SVXPK2  2025-05-08 08:42:40+00  FULL  STREAM    15/0   40s   12MB  32MB    4.54  0/A021B10  0/B343440  OK
```


Журнал postgres содержит отметки об успешном архивировании WAL сегментов:  
INFO: pg_probackup archive-push WAL file: 0000000F000000000000000A, threads: 1/1, batch: 1/1, compression: zlib  
INFO: pg_probackup archive-push completed successfully, pushed: 1, skipped: 0, time elapsed: 134ms  

а каталог РК - заархивированные файлы:  
ls -l /pgsql/pg_backup/wal/PGPBACKUP1
-rw------- 1 postgres postgres   73386 May  8 08:23 0000000F0000000000000009.gz  
-rw------- 1 postgres postgres 1872271 May  8 08:41 0000000F000000000000000A.gz  