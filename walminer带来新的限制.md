# 		[转]walminer带来新的限制

## 原文

XlogMiner是从PostgreSQL的WAL(write ahead logs)日志中解析出执行的SQL语句的工具，并能生成出对应的undo SQL语句。其开源项目地址为：[https://github.com/HighgoSoftware/XLogMiner。](https://github.com/HighgoSoftware/XLogMiner%E3%80%82) 此版本使用限制较大，需要将wal级别设置为logical，而且需要将表设置为IDENTITY FULL模式。这会加剧wal日志的膨胀，降低数据库性能。为迎合PG日志名称的改变，现将XlogMiner改名为WalMiner。新的开源地址为：<https://gitee.com/movead/XLogMiner>

原文地址:

<https://my.oschina.net/lcc1990/blog/3008819>

<https://github.com/HighgoSoftware/XLogMiner>

<https://gitee.com/movead/pg_lightool>

<https://github.com/digoal/blog/blob/master/201902/20190211_01.md> 

## WalMiner功能增强

1.WalMiner支持解析minimal级别以上的任何wal日志级别。

2.无需将表设置为IDENTITY FULL模式。

3.增加对系统表修改的wal记录的解析。

## WalMiner带来的新的限制

walminer可以完整的解析出给定的wal日志中第一个checkpoint点之后的所有wal记录。第一个checkpoint点之前的delete和update记录可能会解析失败。如果一定需要将此记录解析出来，那么只需要增加更早的wal日志即可。

测试过程:

```
postgres=# select * from pg_switch_wal();
-[ RECORD 1 ]-+----------
pg_switch_wal | 0/801D980

postgres=# select pg_walfile_name(pg_current_wal_lsn());
-[ RECORD 1 ]---+-------------------------
pg_walfile_name | 000000010000000000000009

postgres=# insert into test values(3,'c');
INSERT 0 1
postgres=# update test set name='a' where id=1;
UPDATE 1
postgres=# delete from test where id=2;
DELETE 1
postgres=# checkpoint; 
CHECKPOINT
postgres=# insert into test values(2,'c');
INSERT 0 1
postgres=# update test set name='b' where id=2;
UPDATE 1
postgres=# delete from test where id=2;
DELETE 1
postgres=# select walminer_wal_add('/pgdata/pg_wal/000000010000000000000009');
NOTICE:  Get data dictionary from current database.
-[ RECORD 1 ]----+-------------------
walminer_wal_add | 1 file add success

postgres=# select walminer_start('null','null',0,0);
-[ RECORD 1 ]--+--------------------
walminer_start | walminer sucessful!

postgres=# select * from walminer_contents; 
-[ RECORD 1 ]-----+------------------------------------------------------------------------
-----------------
xid               | 622
virtualxid        | 1
timestamptz       | 2019-04-03 10:39:52.1406+08
record_database   | postgres
record_user       | postgres
record_tablespace | pg_default
record_schema     | public
op_type           | INSERT
op_text           | INSERT INTO "public"."test"() VALUES();
op_undo           | DELETE FROM "public"."test" WHERE "id"=NULL AND "name"=NULL AND ctid = 
'(0,0)';
-[ RECORD 2 ]-----+------------------------------------------------------------------------
-----------------
xid               | 623
virtualxid        | 1
timestamptz       | 2019-04-03 10:40:08.076809+08
record_database   | postgres
record_user       | postgres
record_tablespace | pg_default
record_schema     | public
op_type           | UPDATE
op_text           | UPDATE "public"."test" SET VALUES(NULL) (NOTICE:wal is not enought.);
op_undo           | NULL
-[ RECORD 3 ]-----+------------------------------------------------------------------------
-----------------
xid               | 624
virtualxid        | 1
timestamptz       | 2019-04-03 10:40:19.637156+08
record_database   | postgres
record_user       | postgres
record_tablespace | pg_default
record_schema     | public
op_type           | DELETE
op_text           | DELETE FROM "public"."test" WHERE VALUES(NULL) (NOTICE:wal is not enoug
ht.);
op_undo           | NULL
-[ RECORD 4 ]-----+------------------------------------------------------------------------
-----------------
xid               | 625
virtualxid        | 1
timestamptz       | 2019-04-03 10:41:07.160763+08
record_database   | postgres
record_user       | postgres
record_tablespace | pg_default
record_schema     | public
op_type           | INSERT
op_text           | INSERT INTO "public"."test"("id", "name") VALUES(2, 'c');
op_undo           | DELETE FROM "public"."test" WHERE "id"=2 AND "name"='c' AND ctid = '(0,
9)';
-[ RECORD 5 ]-----+------------------------------------------------------------------------
-----------------
xid               | 626
virtualxid        | 1
timestamptz       | 2019-04-03 10:41:18.954766+08
record_database   | postgres
record_user       | postgres
record_tablespace | pg_default
record_schema     | public
op_type           | UPDATE
op_text           | UPDATE "public"."test" SET "name" = 'b' WHERE "id"=2 AND "name"='c';
op_undo           | UPDATE "public"."test" SET "name" = 'c' WHERE "id"=2 AND "name"='b' AND
 ctid = '(0,10)';
-[ RECORD 6 ]-----+------------------------------------------------------------------------
-----------------
xid               | 627
virtualxid        | 1
timestamptz       | 2019-04-03 10:41:25.983368+08
record_database   | postgres
record_user       | postgres
record_tablespace | pg_default
record_schema     | public
op_type           | DELETE
op_text           | DELETE FROM "public"."test" WHERE "id"=2 AND "name"='b';
op_undo           | INSERT INTO "public"."test"("id", "name") VALUES(2, 'b');


发现该日志文件第一次checkpoint之前的delete和update记录解析不出来，而后面的delete和update解析正常；insert解析后不显示插入的具体记录。

postgres=# 
postgres=# 
postgres=# 
postgres=# 
postgres=# 
postgres=# select walminer_wal_add('/pgdata/pg_wal/000000010000000000000008');
NOTICE:  Get data dictionary from current database.
-[ RECORD 1 ]----+-------------------
walminer_wal_add | 1 file add success

postgres=# select walminer_start('null','null',0,0);
-[ RECORD 1 ]--+--------------------
walminer_start | walminer sucessful!

postgres=# select * from walminer_contents; 
-[ RECORD 1 ]-----+-----------------------------------------------------------------------------------------
xid               | 613
virtualxid        | 1
timestamptz       | 2019-04-03 10:35:33.79952+08
record_database   | postgres
record_user       | postgres
record_tablespace | pg_default
record_schema     | public
op_type           | INSERT
op_text           | INSERT INTO "public"."test"("id", "name") VALUES(1, 'a');
op_undo           | DELETE FROM "public"."test" WHERE "id"=1 AND "name"='a' AND ctid = '(0,1)';
-[ RECORD 2 ]-----+-----------------------------------------------------------------------------------------
xid               | 614
virtualxid        | 1
timestamptz       | 2019-04-03 10:35:37.942925+08
record_database   | postgres
record_user       | postgres
record_tablespace | pg_default
record_schema     | public
op_type           | INSERT
op_text           | INSERT INTO "public"."test"("id", "name") VALUES(2, 'b');
op_undo           | DELETE FROM "public"."test" WHERE "id"=2 AND "name"='b' AND ctid = '(0,2)';
-[ RECORD 3 ]-----+-----------------------------------------------------------------------------------------
xid               | 615
virtualxid        | 1
timestamptz       | 2019-04-03 10:35:42.290945+08
record_database   | postgres
record_user       | postgres
record_tablespace | pg_default
record_schema     | public
op_type           | INSERT
op_text           | INSERT INTO "public"."test"("id", "name") VALUES(3, 'c');
op_undo           | DELETE FROM "public"."test" WHERE "id"=3 AND "name"='c' AND ctid = '(0,3)';
-[ RECORD 4 ]-----+-----------------------------------------------------------------------------------------
xid               | 616
virtualxid        | 1
timestamptz       | 2019-04-03 10:35:56.163302+08
record_database   | postgres
record_user       | postgres
record_tablespace | pg_default
record_schema     | public
op_type           | UPDATE
op_text           | UPDATE "public"."test" SET "name" = 'a1' WHERE "id"=1 AND "name"='a';
op_undo           | UPDATE "public"."test" SET "name" = 'a' WHERE "id"=1 AND "name"='a1' AND ctid = '(0,4)';
-[ RECORD 5 ]-----+-----------------------------------------------------------------------------------------
xid               | 617
virtualxid        | 1
timestamptz       | 2019-04-03 10:36:07.898683+08
record_database   | postgres
record_user       | postgres
record_tablespace | pg_default
record_schema     | public
op_type           | DELETE
op_text           | DELETE FROM "public"."test" WHERE "id"=3 AND "name"='c';
op_undo           | INSERT INTO "public"."test"("id", "name") VALUES(3, 'c');
-[ RECORD 6 ]-----+-----------------------------------------------------------------------------------------
xid               | 618
virtualxid        | 1
timestamptz       | 2019-04-03 10:36:37.00892+08
record_database   | postgres
record_user       | postgres
record_tablespace | pg_default
record_schema     | public
op_type           | INSERT
op_text           | INSERT INTO "public"."test"("id", "name") VALUES(3, 'd');
op_undo           | DELETE FROM "public"."test" WHERE "id"=3 AND "name"='d' AND ctid = '(0,5)';
-[ RECORD 7 ]-----+-----------------------------------------------------------------------------------------
xid               | 619
virtualxid        | 1
timestamptz       | 2019-04-03 10:36:49.516463+08
record_database   | postgres
record_user       | postgres
record_tablespace | pg_default
record_schema     | public
op_type           | UPDATE
op_text           | UPDATE "public"."test" SET "name" = 'b1' WHERE "id"=2 AND "name"='b';
op_undo           | UPDATE "public"."test" SET "name" = 'b' WHERE "id"=2 AND "name"='b1' AND ctid = '(0,6)';
-[ RECORD 8 ]-----+-----------------------------------------------------------------------------------------
xid               | 620
virtualxid        | 1
timestamptz       | 2019-04-03 10:36:56.565541+08
record_database   | postgres
record_user       | postgres
record_tablespace | pg_default
record_schema     | public
op_type           | DELETE
op_text           | DELETE FROM "public"."test" WHERE "id"=3 AND "name"='d';
op_undo           | INSERT INTO "public"."test"("id", "name") VALUES(3, 'd');

加入上一个wal日志，再次解析，发现所有记录解析正常。
```

