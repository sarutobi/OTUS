# Разворачиваем и настраиваем БД с большими данными

## Цель:

- знать различные механизмы загрузки данных

- уметь пользоваться различными механизмами загрузки данных

Необходимо провести сравнение скорости работы запросов на различных СУБД
1 Выбрать одну из СУБД
2 Загрузить в неё данные (10 Гб)
3 Сравнить скорость выполнения запросов на PosgreSQL и выбранной СУБД
4 Описать что и как делали и с какими проблемами столкнулись

> ВЫполнил загрузку данных из набора taxi_trips (50 ГБ) в Postgresql 13. Запрос 

```
SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) as tips_percent, count(*) as c
FROM taxi_trips
group by payment_type
order by 3;
```
> в Google BigQuery выполнялся 2.5 секунды. Аналогичный запрос в Postrgresql выполнялся 1868 секунд.

> Проведена оптимизация таблицы

```
create index pt_tips_trip_total_idx on taxi_trips(payment_type, tips, trip_total)
```

> После создания индекса тот же самый запрос выполняется 58 секунд.
>
> Настройки PostgreSQL:

```
max_connections = 20
shared_buffers = 4GB
effective_cache_size = 12GB
maintenance_work_mem = 2GB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 52428kB
min_wal_size = 4GB
max_wal_size = 8GB
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_workers = 4
maintenance_work_mem = 6GB
```
