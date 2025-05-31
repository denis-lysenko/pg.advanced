# Домашнее задание 8

## Использование Managed Service for PostgreSQL 

Задание выполняется в ОС Linux

Установим YC CLI

``` console
curl https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash -s -- -h
```

Создадим профиль YC

``` console
yc config profile create main
```

Инициализируем окружение YC CLI

``` console
yc init
```

Создадим новую группу безопасности для доступа с текущего хоста. Публичный IP определим запросом на ipecho.net

``` console
yc vpc security-group create   \
--name pda-managed-pg   \
--rule "direction=ingress,port=6432,protocol=tcp,v4-cidrs=[$(curl -s ipecho.net/plain)/32]"   \
--network-name default

id: enp08mg*******
folder_id: b1gts8p4********
created_at: "2025-05-31T05:23:11Z"
name: pda-managed-pg
network_id: enprro3o4********
status: ACTIVE
rules:
  - id: enpi7vf*********
    direction: INGRESS
    ports:
      from_port: "6432"
      to_port: "6432"
    protocol_name: TCP
    protocol_number: "6"
    cidr_blocks:
      v4_cidr_blocks:
        - 178.*.*.28/32
```

Создадим управляемый экземпляр с пресетом в одной зоне (для экономии гранта)

``` console
yc managed-postgresql cluster create \
--name pga-managed-pg1 \
  --environment production \
  --network-name default \
  --host zone-id=ru-central1-a,assign-public-ip=true\
  --resource-preset c3-c2-m4 \
  --security-group-ids enp08mg******** \
  --postgresql-version 16  \
  --disk-size 10 \
  --disk-type network-hdd \
  --user name=test,password=testsecret \
  --database name=test,owner=test

done (5m59s)
id: c9qgrat1************ 
folder_id: b1gts8p4u***********
created_at: "2025-05-31T06:06:14.963060Z"
name: pga-managed-pg1
environment: PRODUCTION
```

Список кластеров

``` console
yc managed-postgresql cluster list
+----------------------+-----------------+---------------------+--------+---------+
|          ID          |      NAME       |     CREATED AT      | HEALTH | STATUS  |
+----------------------+-----------------+---------------------+--------+---------+
| c9qgrat1************ | pga-managed-pg1 | 2025-05-31 06:06:14 | ALIVE  | RUNNING |
+----------------------+-----------------+---------------------+--------+---------+
```

Получим CA сертификат YC

``` console
mkdir -p ~/.postgresql && \
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
     --output-document ~/.postgresql/root.crt && \
chmod 0655 ~/.postgresql/root.crt
```

Узнаем FQDN созданного хоста

``` console
yc managed-postgresql host list --cluster-name pga-managed-pg1 --format json|jq .[].name
"rc1a-n54a7s7*******.mdb.yandexcloud.net"
```

Подключимся и получим список БД

``` console
root@astrase1:~# psql -h "rc1a-n54a7s7*******.mdb.yandexcloud.net" -p 6432 -d test  -U test                                            Пароль пользователя test:
psql (15.12, сервер 16.8 (Ubuntu 16.8-201-yandex.57408.77788938eb))
ПРЕДУПРЕЖДЕНИЕ: psql имеет базовую версию 15, а сервер - 16.
                Часть функций psql может не работать.
SSL-соединение (протокол: TLSv1.3, шифр: TLS_AES_256_GCM_SHA384, сжатие: выкл.)
Введите "help", чтобы получить справку.

test=> \l
                                                Список баз данных
    Имя    | Владелец | Кодировка | LC_COLLATE | LC_CTYPE | локаль ICU | Провайдер локали |     Права доступа
-----------+----------+-----------+------------+----------+------------+------------------+-----------------------
 postgres  | postgres | UTF8      | C          | C        | en-US      | icu              |
 template0 | postgres | UTF8      | C          | C        | en-US      | icu              | =c/postgres          +
           |          |           |            |          |            |                  | postgres=CTc/postgres
 template1 | postgres | UTF8      | C          | C        | en-US      | icu              | =c/postgres          +
           |          |           |            |          |            |                  | postgres=CTc/postgres
 test      | test     | UTF8      | C          | C        | en-US      | icu              | =T/test              +
           |          |           |            |          |            |                  | test=CTc/test        +
           |          |           |            |          |            |                  | admin=c/test         +
           |          |           |            |          |            |                  | monitor=c/test       +
           |          |           |            |          |            |                  | mdb_odyssey=c/test   +
           |          |           |            |          |            |                  | postgres=c/test
(4 строки)
```
