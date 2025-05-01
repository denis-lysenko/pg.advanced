# Домашнее задание 1

## Работа с уровнями изоляции транзакции в PostgreSQL

### 1. Создание проекта

Создана ВМ  в YC. Выполнена установка обновлений и перезагрузка.

### 2. Настройка SSH-доступа

Была сгенерирована RSA ключевая пара. Для доступа к ВМ по SSH был установлен ключ
den@:~$ cat ~/.ssh/authorized_keys
ssh-rsa AAAAB3Nza skipped IpOE= den@N4

### 3. Установка PostgreSQL

```console
den@:~$ apt install postgresql
```

Зададим переменные и алиасы для удобства:
Каталог исполняемых файлов лежит

```console
pg_config --bindir
/usr/lib/postgresql/16/bin
```

Данные лежат

```console
sudo -Eupostgres /usr/lib/postgresql/16/
bin/psql -tA -c "show data_directory;" 
/var/lib/postgresql/16/main
#или sudo -u postgres pg_lsclusters
```

Скорректируем переменные и создадим алиасы для работы:

```console
sudo sh -c  'echo "export PATH=/usr/lib/postgresql/16/bin:$PATH
export PGDATA=/var/lib/postgresql/16/main" >/etc/profile.d/postgres.sh'
sudo chmod 644 /etc/profile.d/postgres.sh
. /etc/profile.d/postgres.sh

echo "alias ppsql='sudo -Eupostgres psql'" >>~/.bashrc
. ~/.bashrc
```

### 4. Подключение к PostgreSQL

```console
den@:~$ppsql
psql (16.8 (Ubuntu 16.8-0ubuntu0.24.04.1))
Type "help" for help.

postgres=#
```

### 5. Работа с транзакциями

Создадим БД для работы:

```sql
postgres=# create database hw1;
CREATE DATABASE
postgres=# \c hw1
You are now connected to database "hw1" as user "postgres".
hw1=#
```

Заполним таблицу

```sql
hw1=# create table shipments(id serial, product_name text, quantity int, destination text);
insert into shipments(product_name, quantity, destination) values('bananas', 1000, 'Europe');
insert into shipments(product_name, quantity, destination) values('coffee', 500, 'USA');
commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1
```

### 6. Изучение уровней изоляции

Текущий уровень изоляции:  

```sql
hw1=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
```

Начинаем новую транзакцию и добавляем запись:
Сессия 1 и 2:

```sql
hw1=# begin transaction ;
BEGIN
```

Сессия 1:

```sql
hw1=*# insert into shipments(product_name, quantity, destination) values('sugar', 300, 'Asia');
INSERT 0 1
hw1=*#
```

Проверяем видимость во второй сессии - запись не видна:
Сессия 2:

```sql
hw1=*# select * from shipments;
 id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
(2 rows)
```

> Транзакция еще не была зафиксирована (COMMIT), поэтому ни при каких уровнях изоляции (в том числе Read uncommitted, т.к. изоляция использует механизм снимков) не может быть видна в другой сессии.

Фиксируем транзакцию:
Сессия 1:

```sql
hw1=*# commit;
COMMIT
```

Данные видны во второй сессии:
Сессия 2:

```sql
hw1=*# select * from shipments;
 id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
  3 | sugar        |      300 | Asia
(3 rows)
```

Транзакция была зафиксирована, поэтому данные видны в другой сессии.

### 7. Эксперименты с уровнем изоляции Repeatable Read

Сессия 1 и 2:

```sql
hw1=# begin transaction ;                        
BEGIN
hw1=*# set transaction isolation level repeatable read;
SET
hw1=*#
```

Добавляем данные:
Сессия 1:

```sql
hw1=*# insert into shipments(product_name, quantity, destination) values('bananas', 2000, 'Africa');
INSERT 0 1
```

Проверяем видимость во второй сессии - запись не видна:
Сессия 2:

```sql
hw1=*# select * from shipments;
 id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
  3 | sugar        |      300 | Asia
(3 rows)
```

> Принципиального отличия в доступе к незафиксированным транзакциям у уровня RR от предыдущего уровня нет.

Фиксируем транзакцию:
Сессия 1:

```sql
hw1=*# commit;
COMMIT
```

Данные все еще не видны:

```sql
hw1=*# select * from shipments;                  
id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
  3 | sugar        |      300 | Asia
(3 rows)
```

> Этот уровень изоляции решает проблему неповторяющегося чтения с помощью снимка, который формируется в начале транзакции, в отличие от уровня RC, где он формируется каждым оператором.
Пока не произошла фиксация транзакции в Сессии 2 данных мы не увидим.

Фиксируем транзакцию Сессии 2 и проверяем:

```sql
hw1=*# commit;
COMMIT
hw1=# select * from shipments;
 id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
  3 | sugar        |      300 | Asia
  4 | bananas      |     2000 | Africa
(4 rows)
```

Данные видны.
