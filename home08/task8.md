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

> Значения tps в районе 550-600

```
[ 310s ] thds: 10 tps: 564.47 qps: 16552.79 (r/w/o: 7518.37/7782.77/1251.65) lat (ms,95%): 49.21 err/s 63.85 reconn/s: 0.00
[ 320s ] thds: 10 tps: 573.91 qps: 16835.82 (r/w/o: 7664.13/7902.82/1268.87) lat (ms,95%): 47.47 err/s 62.42 reconn/s: 0.00
[ 330s ] thds: 10 tps: 549.63 qps: 16230.72 (r/w/o: 7380.36/7633.06/1217.30) lat (ms,95%): 50.11 err/s 62.13 reconn/s: 0.00
[ 340s ] thds: 10 tps: 550.25 qps: 16263.60 (r/w/o: 7393.44/7646.26/1223.91) lat (ms,95%): 49.21 err/s 64.11 reconn/s: 0.00
[ 350s ] thds: 10 tps: 567.00 qps: 16590.25 (r/w/o: 7527.38/7790.94/1271.92) lat (ms,95%): 48.34 err/s 70.86 reconn/s: 0.00
[ 360s ] thds: 10 tps: 544.29 qps: 16006.77 (r/w/o: 7263.09/7532.43/1211.25) lat (ms,95%): 51.02 err/s 64.03 reconn/s: 0.00
[ 370s ] thds: 10 tps: 553.82 qps: 16399.47 (r/w/o: 7457.63/7715.24/1226.59) lat (ms,95%): 50.11 err/s 62.88 reconn/s: 0.00
```

> Параметры конфигурации:

```
max_connections = 20
shared_buffers = 4GB
effective_cache_size = 12GB
maintenance_work_mem = 1GB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 104857kB
min_wal_size = 2GB
max_wal_size = 8GB
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_workers = 4
max_parallel_maintenance_workers = 2
fsync=off
autovacuum=off
checkpoint_timeout='1h'
```
