# Работа с журналами

## Цель:

- уметь работать с журналами и контрольными точками

- уметь настраивать параметры журналов

## Выполнение

1. Настройте выполнение контрольной точки раз в 30 секунд.

    alter system set checkpoint_timeout=30;
    select pg_reload_conf();

2. 10 минут c помощью утилиты pgbench подавайте нагрузку.

    pgbench -c8 -P60 -T600 -U postgres postgres

3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.

    du -sh /var/lib/postgresql/12/main/pg_wal/
    275M
    481M

    select (481-275)/20.0;
     10.3000000000000000

4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?

    select checkpoints_timed, checkpoints_req from pg_stat_bgwriter;

    checkpoints_timed | checkpoints_req

                  305 |               2

    checkpoints_timed | checkpoints_req 
    
                  326 |               2

> Всего была 21 контрольная точка, ожидалось 20. Несовпадение связано с несовпадением времени начала/окончания теста и временем создания контрольной точки.

5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.

    в синхронном режиме

    progress: 120.0 s, 740.5 tps, lat 10.760 ms stddev 7.882

    progress: 180.0 s, 750.4 tps, lat 10.617 ms stddev 7.524

    progress: 240.0 s, 757.2 tps, lat 10.522 ms stddev 7.528

    progress: 300.0 s, 747.3 tps, lat 10.663 ms stddev 7.663

    progress: 360.0 s, 716.6 tps, lat 11.120 ms stddev 8.070

    в асинхронном режиме
    progress: 120.0 s, 1680.2 tps, lat 4.676 ms stddev 2.321

    progress: 180.1 s, 900.4 tps, lat 8.699 ms stddev 23.523

    progress: 240.1 s, 926.0 tps, lat 8.508 ms stddev 23.398

    progress: 300.1 s, 942.6 tps, lat 8.334 ms stddev 23.170

    progress: 360.1 s, 948.2 tps, lat 8.275 ms stddev 23.110

> Запись данных в асинхронном режиме проходит быстрее (отсутствует задержка на подтверждение записи).

6. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?

    /usr/lib/postgresql/12/bin/pg_checksums -e -D /var/lib/postgresql/12/main/

    /usr/lib/postgresql/12/bin/pg_checksums -e -D /var/lib/postgresql/12/main/

    WARNING:  page verification failed, calculated checksum 56320 but expected 41

    ERROR:  invalid page in block 0 of relation base/21361/22967

> После изменения данных внутри файла таблицы не совпадает контрольная сумма блока (данные повреждены). Наиболее правильный вариант восстановить кластер из бэкапа. Если такой возможности нет, то можно изменить параметр

    alter system set ignore_checksum_failure=on;

> и повторить запрос.
