# Секционирование таблицы

Секционировать большую таблицу из демо базы flights

## Выполнение

- создана виртуальная машина otus-flights

- развернута база данных demo с данными за 12 месяцев

- создана секционированная таблица

    ```sql
    create table flights2(like flights) partition by range(scheduled_departure);
    ```
- созданы секции таблицы для хранения данных в архиве и оперативных данных

    ```sql
    create table flights2_201709 partition of flights2 for values from ('2017-09-01') to ('2017-10-01');
    create table flights2_201708 partition of flights2 for values from ('2017-08-01') to ('2017-09-01');
    create table flights2_201707 partition of flights2 for values from ('2017-07-01') to ('2017-08-01');
    create table flights2_201706 partition of flights2 for values from ('2017-06-01') to ('2017-07-01');
    create table flights2_storage partition of flights2 for values from ('2016-01-01') to ('2017-06-01');
    ```

- При выборке данных за месяц из секционированной таблицы по датам выборка идет только из секции:

    ```
    demo=# explain analyse select * from flights2 where scheduled_departure >= '2017-08-29' and scheduled_departure < '2017-08-30';
                                                                                   QUERY PLAN                                                                               
    ------------------------------------------------------------------------------------------------------------------------------------------------------------------------
     Seq Scan on flights2_201708 flights2  (cost=0.00..451.52 rows=519 width=63) (actual time=0.015..4.328 rows=546 loops=1)
       Filter: ((scheduled_departure >= '2017-08-29 00:00:00+00'::timestamp with time zone) AND (scheduled_departure < '2017-08-30 00:00:00+00'::timestamp with time zone))
       Rows Removed by Filter: 16289
     Planning Time: 0.169 ms
     Execution Time: 4.398 ms
    (5 rows)

    ```

- при выборке данных из оригинальной таблицы сканируется вся таблица целиком:

    ```
    demo=# explain analyse select * from flights where scheduled_departure >= '2017-08-29' and scheduled_departure < '2017-08-30';
                                                                                          QUERY PLAN
    ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
     Gather  (cost=1000.00..5575.39 rows=555 width=63) (actual time=0.331..43.766 rows=546 loops=1)
       Workers Planned: 1
       Workers Launched: 1
       ->  Parallel Seq Scan on flights  (cost=0.00..4519.89 rows=326 width=63) (actual time=0.028..27.500 rows=273 loops=2)
         Filter: ((scheduled_departure >= '2017-08-29 00:00:00+00'::timestamp with time zone) AND (scheduled_departure < '2017-08-30 00:00:00+00'::timestamp with time zone))
         Rows Removed by Filter: 107160
     Planning Time: 0.252 ms
     Execution Time: 43.873 ms
    (8 rows)
    ```

## Секционирование таблицы с помощью наследования и правил (rules)

    ```sql
    CREATE TABLE flights3(LIKE flights);
    CREATE TABLE flights3_201709(
      CHECK(scheduled_departure >= '2017-09-01' AND scheduled_departure < '2017-10-01')
    ) INHERITS(flights3);
    CREATE TABLE flights3_201708(
      CHECK(scheduled_departure >= '2017-08-01' AND scheduled_departure < '2017-09-01')
    ) INHERITS(FLIGHTS3);
    CREATE TABLE flights3_201707(
      CHECK(scheduled_departure >= '2017-07-01' AND scheduled_departure < '2017-08-01')
    ) INHERITS(flights3);
    CREATE TABLE flights3_201706(
      CHECK(scheduled_departure >= '2017-06-01' AND scheduled_departure < '2017-07-01')
    ) INHERITS(flights3);
    CREATE TABLE flights3_storage(
	CHECK(scheduled_departure >= '2016-01-01' AND scheduled_departure < '2017-06-01')
    ) INHERITS(flights3);

    create rule flights3_insert_storage as
      on insert to flights3 where(scheduled_departure >= '2016-01-01' AND scheduled_departure < '2017-06-01')
      do instead insert into flights3_storage values(new.*);
    create rule flights3_insert_201706 as
      on insert to flights3 where(scheduled_departure >= '2017-06-01' AND scheduled_departure < '2017-07-01')
      do instead insert into flights3_201706 values(new.*);
    create rule flights3_insert_201707 as
      on insert to flights3 where(scheduled_departure >= '2017-07-01' AND scheduled_departure < '2017-08-01')
      do instead insert into flights3_201707 values(new.*);
    create rule flights3_insert_201708 as
      on insert to flights3 where(scheduled_departure >= '2017-08-01' AND scheduled_departure < '2017-09-01')
      do instead insert into flights3_201708 values(new.*);
    create rule flights3_insert_201709 as
      on insert to flights3 where(scheduled_departure >= '2017-09-01' AND scheduled_departure < '2017-10-01')
      do instead insert into flights3_201709 values(new.*);
    ```

- Проверка работы секционирования с помощью наследования

    ```
    demo=# explain analyse select * from flights3 where scheduled_departure >= '2017-08-29' and scheduled_departure < '2017-08-30';
                                                                                      QUERY PLAN
    ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
     Append  (cost=0.00..454.12 rows=520 width=63) (actual time=0.016..4.023 rows=546 loops=1)
       ->  Seq Scan on flights3 flights3_1  (cost=0.00..0.00 rows=1 width=170) (actual time=0.002..0.003 rows=0 loops=1)
             Filter: ((scheduled_departure >= '2017-08-29 00:00:00+00'::timestamp with time zone) AND (scheduled_departure < '2017-08-30 00:00:00+00'::timestamp with time zone))
       ->  Seq Scan on flights3_201708 flights3_2  (cost=0.00..451.52 rows=519 width=63) (actual time=0.013..3.940 rows=546 loops=1)
             Filter: ((scheduled_departure >= '2017-08-29 00:00:00+00'::timestamp with time zone) AND (scheduled_departure < '2017-08-30 00:00:00+00'::timestamp with time zone))
         Rows Removed by Filter: 16289
     Planning Time: 0.971 ms
     Execution Time: 4.103 ms
    (8 rows)
    ```

