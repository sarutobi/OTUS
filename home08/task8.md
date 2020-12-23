# Нагрузочное тестирование и тюнинг PostgreSQL

## Цель:

- делать нагрузочное тестирование PostgreSQL

- настраивать параметры PostgreSQL для достижения максимальной производительности

## Выполнение

- сделать проект <firstname>-<lastname>-<yyyymmdd>-10

>Создан проект Valery-20201212

- сделать инстанс Google Cloud Engine типа e2-medium с ОС Ubuntu 20.04

> Инстанс создан

- поставить на него PostgreSQL 13 из пакетов собираемых postgres.org

> Postgresql 13 установлен

- настроить кластер PostgreSQL 13 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины

> Настроено с помощью онлайн-конфигуратора https://pgtune.leopard.in.ua/#/
> Дополнительно к настройкам конфигуратора добавлен параметр fsync off

- нагрузить кластер через утилиту https://github.com/Percona-Lab/sysbench-tpcc (требует установки https://github.com/akopytov/sysbench)

> Выполнена установка tpcc, запущены подготовка базы

```
./tpcc.lua --db-driver=pgsql --pgsql-user=config --pgsql-db=config --pgsql-password=*** --time=600 --threads=10 --report-interval=10 --tables=10 prepare
```

> и затем тестирование

```
./tpcc.lua --db-driver=pgsql --pgsql-user=config --pgsql-db=config --pgsql-password=*** --time=600 --threads=10 --report-interval=10 --tables=10 --scale=1 run
```

- написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему 

> Значения tps в середине теста были в районе 300(при загрузке процессора в 150%) после чего стабилизировались на 145-160

```
[ 130s ] thds: 10 tps: 248.47 qps: 7305.10 (r/w/o: 3315.20/3436.94/552.97) lat (ms,95%): 108.68 err/s 29.16 reconn/s: 0.00
[ 140s ] thds: 10 tps: 273.84 qps: 8228.11 (r/w/o: 3734.74/3877.87/615.50) lat (ms,95%): 99.33 err/s 34.96 reconn/s: 0.00
[ 150s ] thds: 10 tps: 294.61 qps: 8502.11 (r/w/o: 3856.90/3991.50/653.72) lat (ms,95%): 97.55 err/s 34.10 reconn/s: 0.00
[ 160s ] thds: 10 tps: 294.70 qps: 8634.35 (r/w/o: 3925.38/4046.08/662.90) lat (ms,95%): 92.42 err/s 37.80 reconn/s: 0.00
[ 170s ] thds: 10 tps: 302.69 qps: 8986.15 (r/w/o: 4080.84/4233.34/671.97) lat (ms,95%): 92.42 err/s 34.20 reconn/s: 0.00
[ 180s ] thds: 10 tps: 300.92 qps: 8992.26 (r/w/o: 4087.25/4236.96/668.04) lat (ms,95%): 92.42 err/s 33.80 reconn/s: 0.00
[ 190s ] thds: 10 tps: 314.31 qps: 9127.83 (r/w/o: 4137.45/4288.42/701.96) lat (ms,95%): 89.16 err/s 38.72 reconn/s: 0.00
...
[ 490s ] thds: 10 tps: 134.63 qps: 3966.55 (r/w/o: 1804.21/1861.00/301.34) lat (ms,95%): 244.38 err/s 16.63 reconn/s: 0.00
[ 500s ] thds: 10 tps: 155.39 qps: 4611.55 (r/w/o: 2094.49/2173.56/343.51) lat (ms,95%): 231.53 err/s 16.77 reconn/s: 0.00
[ 510s ] thds: 10 tps: 150.56 qps: 4468.29 (r/w/o: 2029.90/2108.48/329.92) lat (ms,95%): 231.53 err/s 15.10 reconn/s: 0.00
[ 520s ] thds: 10 tps: 145.64 qps: 4213.18 (r/w/o: 1913.78/1981.10/318.30) lat (ms,95%): 235.74 err/s 14.00 reconn/s: 0.00
[ 530s ] thds: 10 tps: 149.39 qps: 4412.74 (r/w/o: 1997.88/2081.68/333.18) lat (ms,95%): 231.53 err/s 17.50 reconn/s: 0.00
[ 540s ] thds: 10 tps: 145.81 qps: 4224.52 (r/w/o: 1919.80/1981.90/322.82) lat (ms,95%): 240.02 err/s 16.40 reconn/s: 0.00
[ 550s ] thds: 10 tps: 159.90 qps: 4646.69 (r/w/o: 2111.99/2183.89/350.80) lat (ms,95%): 227.40 err/s 16.50 reconn/s: 0.00
[ 560s ] thds: 10 tps: 156.00 qps: 4531.46 (r/w/o: 2058.17/2129.92/343.38) lat (ms,95%): 227.40 err/s 16.59 reconn/s: 0.00
[ 570s ] thds: 10 tps: 156.61 qps: 4671.69 (r/w/o: 2124.65/2200.80/346.24) lat (ms,95%): 227.40 err/s 17.51 reconn/s: 0.00

```

> Параметры конфигурации:

```
fsync=off
autovacuum=off
checkpoint_timeout='1h'
max_connections = 20
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 256MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 52428kB
min_wal_size = 2GB
max_wal_size = 8GB
max_worker_processes = 2
max_parallel_workers_per_gather = 1
max_parallel_workers = 2
max_parallel_maintenance_workers = 1
```
