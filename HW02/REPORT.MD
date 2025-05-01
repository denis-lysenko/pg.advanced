# Домашнее задание 2

## 1. Создайте инстанс с Ubuntu 20.04 в Яндекс.Облаке или аналогах

Используется ВМ из ДЗ1 в YC

## 2. Установите Docker Engine

Обновим сертфикаты и ключи репозиориев

```console
den@:~$sudo apt-get install ca-certificates curl
den@:~$sudo install -m 0755 -d /etc/apt/keyrings
den@:~$sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
den@:~$sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Добавим репозиторий docker и выполним установку

```console
den@:~$echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
den@:~$sudo apt-get update
den@:~$sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Проверяем

```console
den@:~$docker --version
Docker version 28.1.1, build 4eba377
```

## 3. Создайте каталог /var/lib/postgres для хранения данных

```console
den@:~$sudo mkdir /var/lib/postgres
```

## 4. Разверните контейнер с PostgreSQL 14, смонтировав в него /var/lib/postgres

Скачиваем требуемый образ

```console
den@:~$sudo docker pull postgres:14
14: Pulling from library/postgres
254e724d7786: Pull complete
38595a9c8160: Pull complete
...
```

Создаем сеть для подключений и запускаем, пробрасывая порт (потребуется далее)

```console
den@:~$sudo docker network create -d bridge postgres
den@:~$sudo docker run -d \
	--name hw2-pg1 \
	-e POSTGRES_PASSWORD=Password \
	-e PGDATA=/var/lib/postgresql/data/pgdata \
	-v /var/lib/postgres:/var/lib/postgresql/data \
	--network postgres \
	-p 5432:5432 \
	postgres:14
den@:~$5f3b42b5a7afb318c49d6a30cc50b31a34a3543eb035f38e8c8fe69af2f2be21
```

Добавим алиас для удобства

```console
echo "alias d='sudo docker'">>~/.bashrc
. ~/.bashrc
```

Смотрим состояние контейнера

```console
den@:~$d ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS      NAMES
5f3b42b5a7af   postgres:14   "docker-entrypoint.s…"   1 minute ago   Up 1 minute   5432/tcp   hw2-pg1
```

## 5. Разверните контейнер с клиентом PostgreSQL

Не нашел готового образа, которому можно доверять. Соберем свой.

```console
den@:~$echo "FROM alpine:3.15
RUN apk --no-cache add postgresql14-client
ENTRYPOINT [ \"psql\" ]">PgClientDockerfile

den@:~$sudo docker build -t mypsql -f PgClientDockerfile .
```

Зададим переменные для подключения:

```console
echo "PGDATABASE=postgres
PGHOST=hw2-pg1
PGPORT=5432
PGUSER=postgres
PGPASSWORD=Password">>~/.env
```

Запустим

```console
den@:~$sudo docker run --rm -it --network postgres --env-file ~/.env mypsql
psql (14.10, server 14.17 (Debian 14.17-1.pgdg120+1))
Type "help" for help.

postgres=#
```

## 6. Подключитесь из контейнера с клиентом к контейнеру с сервером и создайте таблицу с данными о перевозках
Создадим БД и наполним данными

```console
postgres=# create database hw2;
CREATE DATABASE
postgres=# \c hw2
psql (14.10, server 14.17 (Debian 14.17-1.pgdg120+1))
You are now connected to database "hw2" as user "postgres".
hw2=#create table shipments(id serial, product_name text, quantity int, destination text);
insert into shipments(product_name, quantity, destination) values('bananas', 1000, 'Europe');
insert into shipments(product_name, quantity, destination) values('bananas', 1500, 'Asia');
insert into shipments(product_name, quantity, destination) values('bananas', 2000, 'Africa');
insert into shipments(product_name, quantity, destination) values('coffee', 500, 'USA');
insert into shipments(product_name, quantity, destination) values('coffee', 700, 'Canada');
insert into shipments(product_name, quantity, destination) values('coffee', 300, 'Japan');
insert into shipments(product_name, quantity, destination) values('sugar', 1000, 'Europe');
insert into shipments(product_name, quantity, destination) values('sugar', 800, 'Asia');
insert into shipments(product_name, quantity, destination) values('sugar', 600, 'Africa');
insert into shipments(product_name, quantity, destination) values('sugar', 400, 'USA');
hw2=# select count(*) from shipments;
 count
-------
    10
```

## 7. Подключитесь к контейнеру с сервером с ноутбука или компьютера

Подключаемся с win платформы:

```console
PS C:\Users\den> "C:\Program Files\PostgreSQL\14\scripts\runpsql.bat"
C:\Program Files\PostgreSQL\14\scripts\runpsql.bat
PS C:\Users\den> cd "C:\Program Files\PostgreSQL\14\scripts\"
PS C:\Program Files\PostgreSQL\14\scripts> .\runpsql.bat
Server [localhost]: [skipped]
Database [postgres]:
Port [5432]:
Username [postgres]:
Пароль пользователя postgres:
psql (14.4, сервер 14.17 (Debian 14.17-1.pgdg120+1))
ПРЕДУПРЕЖДЕНИЕ: Кодовая страница консоли (866) отличается от основной
                страницы Windows (1251).
                8-битовые (русские) символы могут отображаться некорректно.
                Подробнее об этом смотрите документацию psql, раздел
                "Notes for Windows users".
Введите "help", чтобы получить справку.

postgres=# \l
                                 ╤яшёюъ срч фрээ√ї
    ╚ь     | ┬ырфхыхЎ | ╩юфшЁютър | LC_COLLATE |  LC_CTYPE  |     ╧Ёртр фюёЄєяр
-----------+----------+-----------+------------+------------+-----------------------
 hw2       | postgres | UTF8      | en_US.utf8 | en_US.utf8 |
 postgres  | postgres | UTF8      | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8      | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |           |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8      | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |           |            |            | postgres=CTc/postgres
(4 ёЄЁюъш)
```
  
Проверяем таблицу:

```console
hw2=# select * from shipments limit 1;
 id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
(1 ёЄЁюър)
```

## 8.Удалите контейнер с сервером и создайте его заново

Удаляем контейнер

```console
den@:~$d ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS                     PORTS                                         NAMES
10808303af51   postgres:14   "docker-entrypoint.s…"   11 minutes ago   Up 11 minutes              0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp   hw2-pg1
2ec36210e4d7   mypsql        "psql"                   33 minutes ago   Exited (0) 9 minutes ago                                                 hw2-pg-cl
den@:~$d stop hw2-pg1
hw2-pg1
den@:~$d rm hw2-pg1
hw2-pg1
den@:~$d ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED          STATUS                      PORTS     NAMES
2ec36210e4d7   mypsql    "psql"    35 minutes ago   Exited (0) 11 minutes ago             hw2-pg-cl
```

Создаем заново:

```console
den@:~$sudo docker run -d --name hw2-pg1 -e POSTGRES_PASSWORD=Password -e PGDATA=/var/lib/postgresql/data/pgdata -v /var/lib/postgres:/var/lib/postgresql/data --network postgres -p 5432:5432 postgres:14
e1f3a7b0e174f578b0e1f49437660c26d4d1b6516ba8b713968999575f56a7ea
```

Проверим данные не интерактивно, для ранообразия:

```console
den@:~$d run --rm --network postgres --env-file ~/.env -it mypsql "-dhw2" "-ctable shipments;"
 id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | bananas      |     1500 | Asia
  3 | bananas      |     2000 | Africa
  4 | coffee       |      500 | USA
  5 | coffee       |      700 | Canada
  6 | coffee       |      300 | Japan
  7 | sugar        |     1000 | Europe
  8 | sugar        |      800 | Asia
  9 | sugar        |      600 | Africa
 10 | sugar        |      400 | USA
(10 rows)
```

---
>Описание приводил кратко, т.к. не указано насколько подробно требуется. Проблем особенных не было, кроме того, что пришлось серверный контейнер пересоздавать - задание делал последовательно и порт изначально не пробросил =)