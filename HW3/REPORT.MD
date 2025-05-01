# Домашнее задание 3

**Создайте виртуальную машину c Ubuntu**
Используется ВМ из ДЗ1 в YC

Устанавливаем PostgreSQL

```console
den@:~$sudo apt install postgresql
```

Провяем что кластер запущен

```concole
den@:~$sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

Заходим из под пользователя postgres в psql и делаем произвольную таблицу с произвольным содержимым

```console
den@:~$sudo -u postgres psql
psql (16.8 (Ubuntu 16.8-0ubuntu0.24.04.1))
Type "help" for help.

postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=#
\q
```

Останавливаем postgres

```console
sudo systemctl stop postgresql
```

Создаем новый standard persistent диск
Диск создан

```console
den@:~$lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda     253:0    0   10G  0 disk
├─vda1  253:1    0  9.4G  0 part /
├─vda14 253:14   0    4M  0 part
└─vda15 253:15   0  600M  0 part /boot/efi
vdb     253:16   0   10G  0 disk
```

Делаем раздел на весь диск

```console
den@:~$sudo fdisk /dev/vdb
skipped
Created a new partition 1 of type 'Linux' and of size 10 GiB.
```

Делаем файловую систему (можно было бы xfs, но сделал ext4 для разнообразия)

```console
den@:~$sudo mkfs.ext4 -b 4096 /dev/vdb1
skipped
Writing superblocks and filesystem accounting information: done
```

Монтируем

```console
den@:~$sudo mkdir /mnt/data
den@:~$sudo bash -c "echo '/dev/vdb1 /mnt/data ext4 defaults,auto,nofail,noatime 0 0' >>/etc/fstab"
den@:~$sudo systemctl daemon-reload
den@:~$sudo mount -a
den@:~$lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda     253:0    0   10G  0 disk
├─vda1  253:1    0  9.4G  0 part /
├─vda14 253:14   0    4M  0 part
└─vda15 253:15   0  600M  0 part /boot/efi
vdb     253:16   0   10G  0 disk
└─vdb1  253:17   0   10G  0 part /mnt/data
```

Выдаем разрешения УЗ postgres

```console
den@:~$chown -R postgres:postgres /mnt/data/
```

Перемещаем дынные кластера

```console
den@:~$sudo mv /var/lib/postgresql/16 /mnt/data
```

Пытаемся запустить

```console
den@:~$sudo -u postgres pg_ctlcluster 16 main start
Error: /var/lib/postgresql/16/main is not accessible or does not exist
```

>Не получилось, сервис ведь не знает где теперь находится PGDATA

Делаем необходимую корректировку конфигурационного файла

```console
den@:~$sed -i "s/^data_directory.*/data_directory = '\/mnt\/data\/16\/main'/g" /etc/postgresql/16/main/postgresql.conf
den@:~$sudo systemctl start postgresql
```

>Успешный запуск

Проверяем содержимое таблицы

```console
den@:/mnt/data/16$sudo -upostgres psql -c "table test;"
 c1
----
 1
(1 row)
```

## Задание * ##

Создана тестовая ВМ testvm в YC

Останавливаем postgres на основной ВМ

```console
den@:/mnt/data/16$sudo systemctl stop postgresql
```

Размонтируем папку и корректируем fstab

```console
den@:/mnt/data/16$sudo umount -lf /mnt/data
```

Переподключаем диск в консоли YC на тестовую ВМ
Диск и файловая система на нем видны на тестовой ВМ

```console
den@testvm-2:~$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda     253:0    0   10G  0 disk
├─vda1  253:1    0  9.4G  0 part /
├─vda14 253:14   0    4M  0 part
└─vda15 253:15   0  600M  0 part /boot/efi
vdb     253:16   0   10G  0 disk
└─vdb1  253:17   0   10G  0 part
```

Выполняем установку пакетов на тестовой ВМ

```console
den@testvm-2:~$ sudo apt install postgresql
```

Останавливаем экземпляр postgres на тестовой ВМ

```console
den@testvm-2:~$ sudo systemctl stop postgresql
```

Удаляем текущий экземпляр (не обязательно) на тестовой ВМ

```console
den@testvm-2:~$ sudo rm -rf /var/lib/postgresql/16/*
```

Монтируем данные с подключенного диска на тестовой ВМ

```console
den@testvm-2:~$ sudo mount /dev/vdb1 /var/lib/postgresql/16
den@testvm-2:~$ ls -l /var/lib/postgresql
```

Предоставляем права УЗ postgres на тестовой ВМ

```console
den@testvm-2:~$ sudo chown postgres:postgres /var/lib/postgresql -R
```

Стартуем экземпляр СУБД на тестовой ВМ

```console
den@testvm-2:~$ sudo systemctl start postgresql
```

Проверяем данные таблицы на тестовой ВМ

```console
den@testvm-2:~$ sudo -upostgres psql -c "table test;"
 c1
----
 1
(1 row)
```
