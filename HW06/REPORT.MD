# Домашнее задание 6

## Углубленный анализ производительности. Профилирование. Оптимизация.  

Для задания создана ВМ в YC с параметрами 2 core, 4GB RAM, 10GB HDD

### 1. Методика оценки производительности:  

Поготовим БД размером ~500МБ

```console
sudo -Eupostgres createdb benchmark
sudo -Eupostgres pgbench -i -s 30 benchmark
```

Тестирование будем выполнять стандатным тестом (5 SELECT,INSERT,UPDATE на транзакцию):

```console
sudo -Eupostgres pgbench -c 10 -P 30 -T 120 benchmark
```

Оценка выполняется по выводимому среднему tps.

### 2. Тест неоптимизированной конфигурации: 

>pgbench (16.8 (Ubuntu 16.8-0ubuntu0.24.04.1))
starting vacuum...end.  
progress: 30.0 s, 522.4 tps, lat 19.086 ms stddev 31.092, 0 failed  
progress: 60.0 s, 319.5 tps, lat 31.207 ms stddev 71.119, 0 failed  
progress: 90.0 s, 306.1 tps, lat 32.740 ms stddev 64.549, 0 failed  
progress: 120.0 s, 328.7 tps, lat 30.325 ms stddev 53.802, 0 failed  
transaction type: <builtin: TPC-B (sort of)>  
...

tps = **368.800156** (without initial connection time)

## 3. Тест оптимизированной конфигурации:

Добавим файл postgresql.auto.conf с типовыми настройками для данных ресурсов (не совсем классическая, но на производительность эти настройки сильно не влияют):

```file
# Общие параметры
max_connections = 100
shared_buffers = '1024 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
effective_cache_size = '3 GB'
effective_io_concurrency = 1 
random_page_cost = 4 # ВМ с HDD
synchronous_commit = off
maintenance_io_concurrency = 1

# Оптимизация контрольной точки:
checkpoint_timeout = '15 min'
checkpoint_completion_target = 0.9
max_wal_size = '1024 MB'
min_wal_size = '512 MB'

# Оптимизация WAL
wal_compression = on
wal_buffers = -1 # auto-tuned by Postgres till maximum of segment size (16MB by default)
wal_recycle = off

# Процесс записи
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 0

# Параллельность
max_worker_processes = 2
max_parallel_workers_per_gather = 1
max_parallel_maintenance_workers = 1
max_parallel_workers = 2
parallel_leader_participation = on
```

Добавим рекомендуемые настройки ядра ОС:

```console
echo "
vm.swappiness = 1
vm.dirty_ratio = 15
vm.overcommit_memory = 2
vm.overcommit_ratio = 100 #CommitLimit = (total_RAM - total_huge_pages) *overcommit_ratio / 100 + total_swap
vm.dirty_background_ratio = 1" >> /etc/sysctl.conf
sysctl -p
systemctl restart postgresql
```

>pgbench (16.8 (Ubuntu 16.8-0ubuntu0.24.04.1))
starting vacuum...end.  
progress: 30.0 s, 4192.7 tps, lat 2.381 ms stddev 0.985, 0 failed  
progress: 60.0 s, 4939.9 tps, lat 2.022 ms stddev 0.650, 0 failed  
progress: 90.0 s, 4945.3 tps, lat 2.020 ms stddev 0.634, 0 failed  
progress: 120.0 s, 5032.6 tps, lat 1.985 ms stddev 0.606, 0 failed  
transaction type: <builtin: TPC-B (sort of)>  
...  

tps = **4778.082816** (without initial connection time)

## 3. Тестируем "опасные" настройки:

```file
autovacuum = off
full_page_writes = off
fsync = off
```

>starting vacuum...end.  
progress: 30.0 s, 4786.6 tps, lat 2.085 ms stddev 0.644, 0 failed  
progress: 60.0 s, 4871.6 tps, lat 2.051 ms stddev 0.668, 0 failed  
progress: 90.0 s, 4906.6 tps, lat 2.036 ms stddev 0.678, 0 failed  
progress: 120.0 s, 4942.0 tps, lat 2.021 ms stddev 0.621, 0 failed  
transaction type: <builtin: TPC-B (sort of)>  
...  

tps = **4877.167165** (without initial connection time)

Tps практически не изменился по сравнению с вариантом оптимальных настроек. Это можно объяснить тем, что настройки "опасного" варанта больше относятся к процессу записи измененных данных, который прекрасно себя чувствует при небольших объемах изменений, задействуя кэш ОС и дисковых массивов.
Скорректируем тестовую команду, добавив ключ -N - это приведет к большему объему изменений:  
*Оптимальная конфигурация:*
>pgbench (16.8 (Ubuntu 16.8-0ubuntu0.24.04.1))  
starting vacuum...end.  
progress: 30.0 s, 6132.5 tps, lat 1.627 ms stddev 18.681, 0 failed  
progress: 60.0 s, 8017.1 tps, lat 1.245 ms stddev 0.516, 0 failed  
progress: 90.0 s, 8076.2 tps, lat 1.236 ms stddev 0.494, 0 failed  
progress: 120.0 s, 8014.4 tps, lat 1.246 ms stddev 0.536, 0 failed  
transaction type: <builtin: simple update>  
...  
tps = **7560.702794** (without initial connection time)  

*"Опасная" конфигурация:*  
>pgbench (16.8 (Ubuntu 16.8-0ubuntu0.24.04.1))  
starting vacuum...end.  
progress: 30.0 s, 7952.3 tps, lat 1.254 ms stddev 0.495, 0 failed  
progress: 60.0 s, 8097.8 tps, lat 1.233 ms stddev 0.498, 0 failed  
progress: 90.0 s, 8031.6 tps, lat 1.243 ms stddev 0.542, 0 failed  
progress: 120.0 s, 8149.0 tps, lat 1.225 ms stddev 0.494, 0 failed  
transaction type: <builtin: simple update>  
...  

tps = **8058.310657** (without initial connection time)  

Выиграли около 5%.  
Здесь надо добавить, что отсутствие синхронного комита, которое задано в оптимальной конфигурации, само по себе является кардинальным фактором для улучшения взаимодействия с дисковой подсистемой, т.к. postgres фактически не ждет синхронного подтверждения фиксации от диска. Отстутствие явного вызова fsync на этом фоне и на фоне отсутствия серьезных объемов смотрится бледновато, но на самом деле может кардинально улучшить производительность под серъезной нагрузкой, создав, тем не менее, и серъезные риски для данных.  
Также надо заметить, что "универсальная" настрока производительности хороша для систем с низкой и средней нагрузкой, но плохо подходит для тяжело нагруженных систем, для которых решения по оптимизации закладываюстя на этапе проектирования аппаратной составляющей. Также в таких случаях используются дополнительные настройки, которые хорошо проявляют себя на больших объемах, как то: включение больших страниц, оптимизация дискового хранения, вынос отдельных табличных пространств в память и т.п.
Еще можно добавить, что стоит делать определенные допущения в отношении результатов, т.к. ВМ располагается на разделяемой инфраструктуре, а не на выделенной и они иногда могут быть неожиданными =)  
