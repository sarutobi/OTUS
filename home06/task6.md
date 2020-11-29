# Механизм блокировок

## Цель: понимать как работает механизм блокировок объектов и строк

## Выполнение

1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
```
alter system set deadlock_timeout=200;
alter system set log_lock_timeout=on;
select pg_reload_conf();

create table t1(v int);
insert into t1 values(1),(2),(3);
```

> Далее открываем две сессии, в одной сессии выполняем:

```
begin;
update t1 set v=10 where v=1;
```

> В другой выполняем

```
begin;
update t1 set v=11 where v=1;
```

> После этого в логе появляются записи:

```
grep lock /var/log/postgresql/postgresql-12-main.log 
2020-11-29 10:33:44.145 UTC [9149] postgres@test_lock STATEMENT:  begin
2020-11-29 10:33:54.680 UTC [9149] postgres@test_lock LOG:  process 9149 still waiting for ShareLock on transaction 16862224 after 200.121 ms
2020-11-29 10:33:54.680 UTC [9149] postgres@test_lock DETAIL:  Process holding the lock: 11569. Wait queue: 9149.
2020-11-29 10:33:54.680 UTC [9149] postgres@test_lock CONTEXT:  while updating tuple (0,1) in relation "t1"
2020-11-29 10:33:54.680 UTC [9149] postgres@test_lock STATEMENT:  update t1 set v=11 where v=1;
```

> После этого закрываем обе транзакции.

2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

```
select pg_backend_pid();
 pg_backend_pid
----------------
           9149
(1 row)

test_lock=# update t1 set v=1 where v=10;
UPDATE 1
test_lock=# select locktype, relation::REGCLASS, virtualxid, transactionid as xid, mode, granted from pg_locks where pid=9149;
   locktype    | relation | virtualxid |   xid    |       mode       | granted
---------------+----------+------------+----------+------------------+---------
 relation      | pg_locks |            |          | AccessShareLock  | t
 relation      | t1       |            |          | RowExclusiveLock | t
 virtualxid    |          | 4/138      |          | ExclusiveLock    | t
 transactionid |          |            | 16862226 | ExclusiveLock    | t
(4 rows)

test_lock=# select locktype, relation::REGCLASS, virtualxid, transactionid as xid, mode, granted from pg_locks where pid=11569;
   locktype    | relation | virtualxid |   xid    |       mode       | granted
---------------+----------+------------+----------+------------------+---------
 relation      | t1       |            |          | RowExclusiveLock | t
 virtualxid    |          | 5/163      |          | ExclusiveLock    | t
 transactionid |          |            | 16862227 | ExclusiveLock    | t
 transactionid |          |            | 16862226 | ShareLock        | f
 tuple         | t1       |            |          | ExclusiveLock    | t

test_lock=# select locktype, relation::REGCLASS, virtualxid, transactionid as xid, mode, granted from pg_locks where pid=15504;
   locktype    | relation | virtualxid |   xid    |       mode       | granted 
---------------+----------+------------+----------+------------------+---------
 relation      | t1       |            |          | RowExclusiveLock | t
 virtualxid    |          | 6/201      |          | ExclusiveLock    | t
 tuple         | t1       |            |          | ExclusiveLock    | f
 transactionid |          |            | 16862228 | ExclusiveLock    | t
(4 rows)
```

> Первая таблица показывает блокировку из первого сеанса, наложена блокировка на строку таблицы t1, также наложены блокировки на виртуальный номер транзакции и номер транзакции. Во второй таблицы ожидается освобождение строки таблицы из первой транзакции, наложены блокировка на номер своей транзации и ожидается освобождение номера первой транзакции. В третьей таблице ожидается освобождение строки из первой транзакции, наложена блокировка своего номера транзакции.

3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

```
test_lock=# begin;
BEGIN
test_lock=# update t1 set v=20 where v=2;
UPDATE 1
test_lock=# update t1 set v=12 where v=1;
```

```
test_lock=# begin
test_lock-# ;
BEGIN
test_lock=# update t1 set v=30 where v=3;
UPDATE 1
test_lock=# update t1 set v=11 where v=1;
```

```
BEGIN
test_lock=# update t1 set v=10 where v=1;
UPDATE 1
test_lock=# update t1 set v=31 where v=3;
ERROR:  deadlock detected
DETAIL:  Process 11569 waits for ShareLock on transaction 16862231; blocked by process 15504.
Process 15504 waits for ShareLock on transaction 16862230; blocked by process 11569.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,3) in relation "t1"
```

> В логах postgresql можно увидеть следующее

```
2020-11-29 10:50:40.717 UTC [15504] postgres@test_lock DETAIL:  Process holding the lock: 11569. Wait queue: 15504.
2020-11-29 10:50:40.717 UTC [15504] postgres@test_lock STATEMENT:  update t1 set v=101 where v=10;
2020-11-29 11:00:42.886 UTC [11569] postgres@test_lock LOG:  process 11569 acquired ShareLock on transaction 16862226 after 629474.729 ms
2020-11-29 11:00:42.886 UTC [11569] postgres@test_lock CONTEXT:  while updating tuple (0,4) in relation "t1"
2020-11-29 11:00:42.886 UTC [11569] postgres@test_lock STATEMENT:  update t1 set v=11 where v=10;
2020-11-29 11:00:42.886 UTC [15504] postgres@test_lock LOG:  process 15504 acquired ExclusiveLock on tuple (0,4) of relation 22971 of database 22970 after 602369.762 ms
2020-11-29 11:00:42.886 UTC [15504] postgres@test_lock STATEMENT:  update t1 set v=101 where v=10;
2020-11-29 11:00:43.086 UTC [15504] postgres@test_lock LOG:  process 15504 still waiting for ShareLock on transaction 16862227 after 200.134 ms
2020-11-29 11:00:43.086 UTC [15504] postgres@test_lock DETAIL:  Process holding the lock: 11569. Wait queue: 15504.
2020-11-29 11:00:43.086 UTC [15504] postgres@test_lock CONTEXT:  while locking tuple (0,5) in relation "t1"
2020-11-29 11:00:43.086 UTC [15504] postgres@test_lock STATEMENT:  update t1 set v=101 where v=10;
2020-11-29 11:01:15.507 UTC [15504] postgres@test_lock LOG:  process 15504 acquired ShareLock on transaction 16862227 after 32620.874 ms
2020-11-29 11:01:15.507 UTC [15504] postgres@test_lock CONTEXT:  while locking tuple (0,5) in relation "t1"
2020-11-29 11:01:15.507 UTC [15504] postgres@test_lock STATEMENT:  update t1 set v=101 where v=10;
2020-11-29 11:07:09.493 UTC [15504] postgres@test_lock LOG:  process 15504 still waiting for ShareLock on transaction 16862230 after 200.112 ms
2020-11-29 11:07:09.493 UTC [15504] postgres@test_lock DETAIL:  Process holding the lock: 11569. Wait queue: 15504.
2020-11-29 11:07:09.493 UTC [15504] postgres@test_lock CONTEXT:  while updating tuple (0,5) in relation "t1"
2020-11-29 11:07:09.493 UTC [15504] postgres@test_lock STATEMENT:  update t1 set v=11 where v=1;
2020-11-29 11:07:56.061 UTC [9149] postgres@test_lock LOG:  process 9149 still waiting for ExclusiveLock on tuple (0,5) of relation 22971 of database 22970 after 200.185 ms
2020-11-29 11:07:56.061 UTC [9149] postgres@test_lock DETAIL:  Process holding the lock: 15504. Wait queue: 9149.
2020-11-29 11:07:56.061 UTC [9149] postgres@test_lock STATEMENT:  update t1 set v=12 where v=1;
2020-11-29 11:08:59.178 UTC [11569] postgres@test_lock LOG:  process 11569 detected deadlock while waiting for ShareLock on transaction 16862231 after 200.186 ms
2020-11-29 11:08:59.178 UTC [11569] postgres@test_lock DETAIL:  Process holding the lock: 15504. Wait queue: .
2020-11-29 11:08:59.178 UTC [11569] postgres@test_lock CONTEXT:  while updating tuple (0,3) in relation "t1"
2020-11-29 11:08:59.178 UTC [11569] postgres@test_lock STATEMENT:  update t1 set v=31 where v=3;
2020-11-29 11:08:59.178 UTC [11569] postgres@test_lock ERROR:  deadlock detected
2020-11-29 11:08:59.178 UTC [11569] postgres@test_lock DETAIL:  Process 11569 waits for ShareLock on transaction 16862231; blocked by process 15504.
2020-11-29 11:08:59.178 UTC [11569] postgres@test_lock HINT:  See server log for query details.
2020-11-29 11:08:59.178 UTC [11569] postgres@test_lock CONTEXT:  while updating tuple (0,3) in relation "t1"
2020-11-29 11:08:59.178 UTC [11569] postgres@test_lock STATEMENT:  update t1 set v=31 where v=3;
2020-11-29 11:08:59.178 UTC [15504] postgres@test_lock LOG:  process 15504 acquired ShareLock on transaction 16862230 after 109885.696 ms
2020-11-29 11:08:59.178 UTC [15504] postgres@test_lock CONTEXT:  while updating tuple (0,5) in relation "t1"
2020-11-29 11:08:59.178 UTC [15504] postgres@test_lock STATEMENT:  update t1 set v=11 where v=1;
2020-11-29 11:08:59.179 UTC [9149] postgres@test_lock LOG:  process 9149 acquired ExclusiveLock on tuple (0,5) of relation 22971 of database 22970 after 63317.334 ms
2020-11-29 11:08:59.179 UTC [9149] postgres@test_lock STATEMENT:  update t1 set v=12 where v=1;
```

> Обнаружен DEADLOCK. 

4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
* Попробуйте воспроизвести такую ситуацию.

> Исходя из статьи https://blog.heroku.com/curious-case-table-locking-update-query такая ситуация возможна при конкурентном обновлении таблицы (наложение лока на таблицу при уже имеющихся локах на строки). Воспроизвести не удалось.
