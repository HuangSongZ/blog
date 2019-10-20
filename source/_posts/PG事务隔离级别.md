---
title: PG������뼶��
date: 2019-10-20 23:51:24
categories:PG����
tags:
- ������뼶��
- MVCC
---


- PostgreSQL ���º�ɾ������ʱ, ������ֱ��ɾ���е�����, ���Ǹ����е�ͷ����Ϣ�е�xmax��infomask���롣�����ύ����µ�ǰ���ݿ⼯Ⱥ������״̬��pg_clog�е������ύ״̬��

- PostgreSQL��汾�������Ʋ���ҪUNDO��ռ䡣

- PostgreSQL��repeatable read���뼶�𲻻���������

## READ UNCOMMITTED ����

```SQL
-- session A
postgres=# begin isolation level READ UNCOMMITTED;
BEGIN
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,3) |  498 |    0 |  1 | A
(2 �м�¼)

postgres=# update tb1 set name='a' where id=1;
UPDATE 1

postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,4) |  500 |    0 |  1 | a
(2 �м�¼)


-- session B
-- û�ж���session A �ĸ���
postgres=# begin isolation level READ UNCOMMITTED;
BEGIN
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,3) |  498 |    0 |  1 | A
(2 �м�¼)

-- session A
postgres=# commit;

-- session B
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,4) |  500 |    0 |  1 | a
(2 �м�¼)

```
��������Կ��Կ�����PostgreSQL��֧��read uncommitted������뼶��


## READ COMMITTED ����
```sql

postgres=# create table tb1(id int, name text);
CREATE TABLE
postgres=# insert into tb1 values(1, 'a');
postgres=# insert into tb1 values(2, 'b');


-- session A
postgres=# begin;
BEGIN
-- a, b�ֱ�������id 495��497����
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,1) |  495 |    0 |  1 | a
 (0,2) |  497 |    0 |  2 | b
(2 �м�¼)


-- session B
postgres=# begin;
BEGIN

-- �ỰB��������飬idΪ498
postgres=# select  txid_current();
 txid_current
--------------
          498
(1 �м�¼)

-- ����id=1��name='A'
postgres=# update tb1 set name='A' where id=1;
UPDATE 1
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,3) |  498 |    0 |  1 | A
(2 �м�¼)

-- session A
-- �ỰA����������B�ĸ��ģ�RC��������id=1��һ��xmax�Ѿ���Ϊ�ỰA������id��˵�����м�¼�Ѿ��������Ϊ498����
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,1) |  495 |  498 |  1 | a
 (0,2) |  497 |    0 |  2 | b
(2 �м�¼)

-- ����B�ύ
-- session B
postgres=# commit;
COMMIT

-- ����A�ܹ���������B�ύ��ĸ������ݣ����Ҹ��м�¼��xmaxΪ0
session A
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,3) |  498 |    0 |  1 | A
(2 �м�¼)


postgres=# select * from heap_page_items(get_raw_page('tb1', 0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |     t_data
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------
  1 |   8160 |        1 |     30 |    495 |    498 |        0 | (0,3)  |       16386 |       1282 |     24 |        |       | \x010000000561
  2 |   8128 |        1 |     30 |    497 |      0 |        0 | (0,2)  |           2 |       2306 |     24 |        |       | \x020000000562
  3 |   8096 |        1 |     30 |    498 |      0 |        0 | (0,3)  |       32770 |      10498 |     24 |        |       | \x010000000541
(3 �м�¼)

```
PostgreSQL ���º�ɾ������ʱ, ������ֱ��ɾ���е�����, ���Ǹ����е�ͷ����Ϣ�е�xmax��infomask���롣

## REPEATABLE READ ����
```SQL
-- session A
postgres=# begin isolation level REPEATABLE READ;
BEGIN
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,4) |  500 |    0 |  1 | a
(2 �м�¼)

-- session B
-- session B �޸�����, ���ύ
postgres=# begin;
BEGIN
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,4) |  500 |    0 |  1 | a
(2 �м�¼)

postgres=# update tb1 set name='A' where id=1;
UPDATE 1
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,5) |  501 |    0 |  1 | A
(2 �м�¼)

postgres=# commit;
COMMIT

-- session A
-- δ���ֲ����ظ�������
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,4) |  500 |  501 |  1 | a
(2 �м�¼)

-- session C �������ݲ��ύ
postgres=# begin;
BEGIN
postgres=# insert into tb1 values(3, 'c');
INSERT 0 1
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,5) |  501 |    0 |  1 | A
 (0,6) |  501 |    0 |  3 | c
(3 �м�¼)

postgres=# commit;
COMMIT

-- session A
-- δ���ֻ����
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,4) |  500 |  501 |  1 | a
(2 �м�¼)

```


## PostgresSQL �ö��龰
PostgresSQL �� REPEATABLE READ ������ֻö�������ͨ�� READ COMMITTED ���Իö�

```
-- session A
postgres=# begin ISOLATION LEVEL READ COMMITTED;
BEGIN
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,5) |  501 |    0 |  1 | A
 (0,6) |  501 |    0 |  3 | c
(3 �м�¼)

-- session B ɾ��һ����¼���ύ
postgres=# begin;
BEGIN
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,5) |  501 |    0 |  1 | A
 (0,6) |  501 |    0 |  3 | c
(3 �м�¼)


postgres=# delete from tb1 where id=1;
DELETE 1
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,6) |  501 |    0 |  3 | c
(2 �м�¼)

postgres=# commit;

-- session A �ö����ոն���ʱ��3����¼��������2����¼��
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,6) |  501 |    0 |  3 | c
(2 �м�¼)


-- session C ����һ����¼
postgres=# begin;
BEGIN
postgres=# insert into tb1 values(4, 'c');
INSERT 0 1
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,6) |  501 |    0 |  3 | c
 (0,7) |  503 |    0 |  4 | c
(3 �м�¼)

postgres=# commit;


-- session A �ٴλö����ոն�ȡ�ļ�¼��û��id=4�ļ�¼��
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,6) |  501 |    0 |  3 | c
 (0,7) |  503 |    0 |  4 | c
(3 �м�¼)

```

## REPEATABLE READ �쳣����

```sql
-- session A
postgres=# begin ISOLATION LEVEL REPEATABLE READ;
BEGIN
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,6) |  501 |    0 |  3 | c
 (0,7) |  503 |    0 |  4 | c
(3 �м�¼)

-- session B ���»���ɾ��id=4��¼, ���ύ.
postgres=# begin;
BEGIN
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,6) |  501 |    0 |  3 | c
 (0,7) |  503 |    0 |  4 | c
(3 �м�¼)

postgres=# delete from tb1 where id=4;
DELETE 1
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,6) |  501 |    0 |  3 | c
(2 �м�¼)

postgres=# commit;

-- session A ���»���ɾ��id=4������¼. �ᱨ��ع�
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,6) |  501 |    0 |  3 | c
 (0,7) |  503 |  504 |  4 | c
(3 �м�¼)

postgres=# update tb1 set name='d' where id=4;
ERROR:  could not serialize access due to concurrent delete


```


## SERIALIZABLE ����
```
-- session A
postgres=# select pg_backend_pid();
 pg_backend_pid
----------------
          24300
(1 �м�¼)

postgres=# truncate tb1;
TRUNCATE TABLE
postgres=# insert into tb1 select generate_series(1,100000);
INSERT 0 100000
postgres=#  begin ISOLATION LEVEL SERIALIZABLE;
BEGIN
postgres=# select sum(id) from tb1 where id=100;
 sum
-----
 100
(1 �м�¼)


-- session B
postgres=# select pg_backend_pid();
 pg_backend_pid
----------------
          10456
(1 �м�¼)


-- session C
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456);
 relation |  locktype  |  pid  |      mode       | granted
----------+------------+-------+-----------------+---------
 tb1      | relation   | 24300 | AccessShareLock | t
          | virtualxid | 24300 | ExclusiveLock   | t
 tb1      | relation   | 24300 | SIReadLock      | t
(3 �м�¼)


-- session B
postgres=# begin ISOLATION LEVEL SERIALIZABLE;
BEGIN
postgres=# select sum(id) from tb1 where id=10;
 sum
-----
  10
(1 �м�¼)


-- session C
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456);
 relation |  locktype  |  pid  |      mode       | granted
----------+------------+-------+-----------------+---------
 tb1      | relation   | 10456 | AccessShareLock | t
          | virtualxid | 10456 | ExclusiveLock   | t
 tb1      | relation   | 24300 | AccessShareLock | t
          | virtualxid | 24300 | ExclusiveLock   | t
 tb1      | relation   | 10456 | SIReadLock      | t
 tb1      | relation   | 24300 | SIReadLock      | t
(6 �м�¼)


-- session A
postgres=# insert into tb1 values(1, 'a');
INSERT 0 1

-- session B
postgres=# insert into tb1 values(2, 'b');
INSERT 0 1

-- session C
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456) order by pid;
 relation |   locktype    |  pid  |       mode       | granted
----------+---------------+-------+------------------+---------
 tb1      | relation      | 10456 | RowExclusiveLock | t
          | virtualxid    | 10456 | ExclusiveLock    | t
          | transactionid | 10456 | ExclusiveLock    | t
 tb1      | relation      | 10456 | SIReadLock       | t
 tb1      | relation      | 10456 | AccessShareLock  | t
 tb1      | relation      | 24300 | SIReadLock       | t
 tb1      | relation      | 24300 | AccessShareLock  | t
 tb1      | relation      | 24300 | RowExclusiveLock | t
          | virtualxid    | 24300 | ExclusiveLock    | t
          | transactionid | 24300 | ExclusiveLock    | t
(10 �м�¼)

-- session A
postgres=# commit;
COMMIT

-- session C
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456) order by pid;
 relation |   locktype    |  pid  |       mode       | granted
----------+---------------+-------+------------------+---------
 tb1      | relation      | 10456 | AccessShareLock  | t
 tb1      | relation      | 10456 | RowExclusiveLock | t
          | virtualxid    | 10456 | ExclusiveLock    | t
          | transactionid | 10456 | ExclusiveLock    | t
 tb1      | relation      | 10456 | SIReadLock       | t
 tb1      | relation      | 24300 | SIReadLock       | t
(6 �м�¼)

-- session B
postgres=# commit;
ERROR:  could not serialize access due to read/write dependencies among transactions
����:  Reason code: Canceled on identification as a pivot, during commit attempt.
��ʾ:  The transaction might succeed if retried.

-- session C
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456) order by pid;
 relation | locktype | pid | mode | granted
----------+----------+-----+------+---------
(0 �м�¼)


```


## ��������������ĳ���
```
-- session A
postgres=# create index idx_tb1 on tb1(id);
CREATE INDEX
postgres=# begin ISOLATION LEVEL SERIALIZABLE;
BEGIN

-- session C
-- ��������������ҳ��
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456) order by pid;
 relation |  locktype  |  pid  |      mode       | granted
----------+------------+-------+-----------------+---------
 idx_tb1  | relation   | 24300 | AccessShareLock | t
 tb1      | relation   | 24300 | AccessShareLock | t
          | virtualxid | 24300 | ExclusiveLock   | t
 idx_tb1  | page       | 24300 | SIReadLock      | t
 tb1      | tuple      | 24300 | SIReadLock      | t
(5 �м�¼)

-- session B
postgres=# begin ISOLATION LEVEL SERIALIZABLE;
BEGIN
postgres=#  select sum(id) from tb1 where id=10;
 sum
-----
  10
(1 �м�¼)

-- session C
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456) order by pid;
 relation |  locktype  |  pid  |      mode       | granted
----------+------------+-------+-----------------+---------
 idx_tb1  | relation   | 10456 | AccessShareLock | t
 tb1      | relation   | 10456 | AccessShareLock | t
          | virtualxid | 10456 | ExclusiveLock   | t
 idx_tb1  | page       | 10456 | SIReadLock      | t
 tb1      | tuple      | 10456 | SIReadLock      | t
          | virtualxid | 24300 | ExclusiveLock   | t
 tb1      | tuple      | 24300 | SIReadLock      | t
 idx_tb1  | page       | 24300 | SIReadLock      | t
 idx_tb1  | relation   | 24300 | AccessShareLock | t
 tb1      | relation   | 24300 | AccessShareLock | t
(10 �м�¼)


-- session A
postgres=# insert into tb1 values(1, 'a');
INSERT 0 1

-- session C
-- ���� 24300 ��� �л�����
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456) order by pid;
 relation |   locktype    |  pid  |       mode       | granted
----------+---------------+-------+------------------+---------
 idx_tb1  | relation      | 10456 | AccessShareLock  | t
 tb1      | relation      | 10456 | AccessShareLock  | t
          | virtualxid    | 10456 | ExclusiveLock    | t
 idx_tb1  | page          | 10456 | SIReadLock       | t
 tb1      | tuple         | 10456 | SIReadLock       | t
 tb1      | relation      | 24300 | RowExclusiveLock | t
          | virtualxid    | 24300 | ExclusiveLock    | t
          | transactionid | 24300 | ExclusiveLock    | t
 tb1      | tuple         | 24300 | SIReadLock       | t
 idx_tb1  | page          | 24300 | SIReadLock       | t
 idx_tb1  | relation      | 24300 | AccessShareLock  | t
 tb1      | relation      | 24300 | AccessShareLock  | t
(12 �м�¼)

-- session B
postgres=# insert into tb1 values(2, 'b');
INSERT 0 1


-- session C
-- -- ���� 10456 ��� �л�����
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456) order by pid;
 relation |   locktype    |  pid  |       mode       | granted
----------+---------------+-------+------------------+---------
 idx_tb1  | relation      | 10456 | AccessShareLock  | t
 tb1      | relation      | 10456 | AccessShareLock  | t
 tb1      | relation      | 10456 | RowExclusiveLock | t
          | virtualxid    | 10456 | ExclusiveLock    | t
          | transactionid | 10456 | ExclusiveLock    | t
 idx_tb1  | page          | 10456 | SIReadLock       | t
 tb1      | tuple         | 10456 | SIReadLock       | t
          | virtualxid    | 24300 | ExclusiveLock    | t
 idx_tb1  | page          | 24300 | SIReadLock       | t
          | transactionid | 24300 | ExclusiveLock    | t
 tb1      | tuple         | 24300 | SIReadLock       | t
 idx_tb1  | relation      | 24300 | AccessShareLock  | t
 tb1      | relation      | 24300 | AccessShareLock  | t
 tb1      | relation      | 24300 | RowExclusiveLock | t
(14 �м�¼)

-- session A
postgres=# commit;
COMMIT

-- session B
-- ����ҳ����ͬһ��, ���ұ�������������. ���Է����˳�ͻ
postgres=# commit;
ERROR:  could not serialize access due to read/write dependencies among transactions
����:  Reason code: Canceled on identification as a pivot, during commit attempt.
��ʾ:  The transaction might succeed if retried.

-- session C
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456) order by pid;
 relation | locktype | pid | mode | granted
----------+----------+-----+------+---------
(0 �м�¼)


-- �������һ�������ֵ����1������ҳ��û������, ����
-- session A
postgres=# begin ISOLATION LEVEL SERIALIZABLE;
BEGIN
postgres=# select sum(id) from tb1 where id=100;
 sum
-----
 100
(1 �м�¼)


-- session B
postgres=# begin ISOLATION LEVEL SERIALIZABLE;
BEGIN
postgres=# select sum(id) from tb1 where id=10;
 sum
-----
  10
(1 �м�¼)


-- session A
postgres=# insert into tb1 values(1, 'a');
INSERT 0 1
postgres=# commit;
COMMIT

-- session B
postgres=# insert into tb1 values(200000, 'c');
INSERT 0 1
postgres=# commit;
COMMIT

```
> ע������:
>
> PostgreSQL �� hot_standby�ڵ㲻֧�ִ���������뼶��, ֻ��֧��read committed��repeatable read���뼶��



## PostgreSQL��汾��������

```
? RR1 tuple-v1 IDLE IN TRANSACTION;
? RC1 tuple-v1 IDLE IN TRANSACTION;
? RC2 tuple-v1 UPDATE -> tuple-v2 COMMIT;
? RR1 tuple-v1 IDLE IN TRANSACTION;
? RC1 tuple-v2 IDLE IN TRANSACTION;
? RR2 tuple-v2 IDLE IN TRANSACTION;
? RC3 tuple-v2 UPDATE -> tuple-v3 COMMIT;
? RR1 tuple-v1 IDLE IN TRANSACTION;
? RR2 tuple-v2 IDLE IN TRANSACTION;
? RC1 tuple-v3 IDLE IN TRANSACTION;
```

PostgreSQL��汾�������Ʋ���ҪUNDO��ռ䡣