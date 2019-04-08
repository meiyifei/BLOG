# [转]使用 pg_repack 解决表和索引膨胀问题

### 原文地址:

https://blog.csdn.net/ctypyb2002/article/details/85007105

一、pg_repack的安装

pg_ repack 是一个可以在线重建表和索引的扩展 它会在数据库中建立一个和需要清理
的目标表一样的临时表，将目标表中的数据 COPY 到临时表，并在临时表上建立与目标表
一样的索引，然后通过重命名的方式用临时表替换目标表

```
wget https://github.com/reorg/pg_repack/archive/ver_1.4.4.zip 

unzip pg_repack-1.4.4.zip

cd  pg_repack-1.4.4

make

make install

注意:可能会提示pg_config没有找到，这个是因为环境变量没有设置好。(需要在root用户下make)

测试

pg_repack是以extension的方式工作的

(1)验证有没有可用的pg_repack
postgres=# select * from pg_available_extensions where name like '%repack%';
   name    | default_version | installed_version |                           comment       
                     
-----------+-----------------+-------------------+-----------------------------------------
---------------------
 pg_repack | 1.4.4           |                   | Reorganize tables in PostgreSQL database
s with minimal locks
(1 row)

(2)创建pg_repack扩展

postgres=# create extension pg_repack; 
CREATE EXTENSION
postgres=# \dn
  List of schemas
  Name  |  Owner   
--------+----------
 public | postgres
 repack | postgres
(2 rows)

[root@Odyssey pg_repack-1.4.4]# which pg_repack
/usr/local/pg10.3/bin/pg_repack
```

二、pg_repack的使用

（1）online vacuum full (include table indexes)

```
create table tmp_t0(co bigint,c1 bigint primary key);
test=# insert into tmp_t0 select g.i::bigint,g.i::bigint from generate_series(1,1000) as g(i);

test=# with tmp_0 as (
 select pc.oid as reloid,
        pn.nspname,
        pc.relname,
        pn.nspname||'.'||pc.relname as schema_relname,
        pc.relkind
   from pg_class pc,
        pg_namespace pn
  where 1=1
    and pc.relnamespace=pn.oid
)
select t0.reloid as tableoid,
       pt.schemaname||'.'||pt.tablename as schema_tablename,
       pg_relation_filepath(pt.schemaname||'.'||pt.tablename),
       pg_table_size(pt.schemaname||'.'||pt.tablename),
       pg_relation_size(pt.schemaname||'.'||pt.tablename),
       pg_total_relation_size(pt.schemaname||'.'||pt.tablename),
       t1.reloid as indexoid,
       pi.schemaname||'.'||pi.indexname,
       pg_relation_filepath(pi.schemaname||'.'||pi.indexname),
       pg_relation_size(pi.schemaname||'.'||pi.indexname),
       pg_indexes_size(pi.schemaname||'.'||pi.tablename)
 from pg_tables pt
      left outer join pg_indexes pi 
                   on pt.schemaname||'.'||pt.tablename = pi.schemaname||'.'||pi.tablename
      left outer join tmp_0 t0
                   on pt.schemaname||'.'||pt.tablename = t0.schema_relname
      left outer join tmp_0 t1
                   on pi.schemaname||'.'||pi.indexname = t1.schema_relname             
where 1=1
  and pt.schemaname='public'
  and pt.tablename='tmp_t0'
;

-[ RECORD 1 ]----------+-------------------
tableoid               | 57452
schema_tablename       | public.tmp_t0
pg_relation_filepath   | base/57393/57452
pg_table_size          | 73728
pg_relation_size       | 49152
pg_total_relation_size | 114688
indexoid               | 57455
?column?               | public.tmp_t0_pkey
pg_relation_filepath   | base/57393/57455
pg_relation_size       | 40960
pg_indexes_size        | 40960

执行vaccum full操作

[postgres@Odyssey ~]$ pg_repack -n -t tmp_t0 -d test
INFO: repacking table "public.tmp_t0"

-[ RECORD 1 ]----------+-------------------
tableoid               | 57452
schema_tablename       | public.tmp_t0
pg_relation_filepath   | base/57393/57508
pg_table_size          | 73728
pg_relation_size       | 49152
pg_total_relation_size | 114688
indexoid               | 57455
?column?               | public.tmp_t0_pkey
pg_relation_filepath   | base/57393/57511
pg_relation_size       | 40960
pg_indexes_size        | 40960


注意：此时文件的位置已经变化了
```

(2)其他的一些用法

```
online repack database
$ pg_repack --wait-timeout 3600 --jobs 10 --no-order -d peiybdb

online repack schema
$ pg_repack --wait-timeout 3600 --jobs 10 --no-order --schema=public -d peiybdb

online repack table and indexes
$ pg_repack --wait-timeout 3600 --no-order --table public.tmp_t0 -d peiybdb

实例:
	test=# update tmp_t0 set c0=c0+10;
	UPDATE 1000

更新后数据库对象的大小
	-[ RECORD 1 ]----------+-------------------
tableoid               | 57452
schema_tablename       | public.tmp_t0
pg_relation_filepath   | base/57393/65608
pg_table_size          | 114688
pg_relation_size       | 90112
pg_total_relation_size | 204800
indexoid               | 57455
?column?               | public.tmp_t0_pkey
pg_relation_filepath   | base/57393/65611
pg_relation_size       | 90112
pg_indexes_size        | 90112

repack表和索引
[postgres@Odyssey ~]$ pg_repack --wait-timeout 3600 --no-order --table public.tmp_t0 -d test --echo

查看repack后数据库对象的大小

-[ RECORD 1 ]----------+-------------------
tableoid               | 57452
schema_tablename       | public.tmp_t0
pg_relation_filepath   | base/57393/65627
pg_table_size          | 73728
pg_relation_size       | 49152
pg_total_relation_size | 114688
indexoid               | 57455
?column?               | public.tmp_t0_pkey
pg_relation_filepath   | base/57393/65630
pg_relation_size       | 40960
pg_indexes_size        | 40960

可以看到差不多减少了一半的空间


online repack only all index
$ pg_repack --only-indexes --table public.tmp_t0 -d peiybdb 

online repack only some index
$ pg_repack --index public.tmp_t0_pk -d peiybdb 
```

三、pg_repack某个表的过程分析

```
[postgres@Odyssey ~]$ pg_repack --wait-timeout 3600 --no-order --table public.tmp_t0 -d test --echo
LOG: (query) SET search_path TO pg_catalog, pg_temp
LOG: (query) SET search_path TO pg_catalog, pg_temp
LOG: (query) select repack.version(), repack.version_sql()
LOG: (query) SET statement_timeout = 0
LOG: (query) SET search_path = pg_catalog, pg_temp, public
LOG: (query) SET client_min_messages = warning
LOG: (query) SELECT t.*, coalesce(v.tablespace, t.tablespace_orig) as tablespace_dest FROM repack.tables t,  (VALUES (quote_ident($1::text))) as v (tablespace) WHERE (relid = $2::regclass) ORDER BY t.relname, t.schemaname
LOG: 	(param:0) = (null)
LOG: 	(param:1) = public.tmp_t0
INFO: repacking table "public.tmp_t0"
LOG: (query) SELECT pg_try_advisory_lock($1, CAST(-2147483648 + $2::bigint AS integer))
LOG: 	(param:0) = 16185446
LOG: 	(param:1) = 57452

使用advisory咨询锁
设置RC事务隔离级别
锁定表为 ACCESS EXCLUSIVE 模式，此时其他session不能对表做任何操作
创建触发器 AFTER INSERT OR DELETE OR UPDATE ON public.tmp_t0 FOR EACH ROW。
将 public.tmp_t0 变化的数据插入到 repack.log_57452 表。(这个日志表就是为了解决由于原表有排它锁而导致这张表被独占锁定)
LOG: (query) BEGIN ISOLATION LEVEL READ COMMITTED
LOG: (query) SET LOCAL statement_timeout = 100
LOG: (query) LOCK TABLE public.tmp_t0 IN ACCESS EXCLUSIVE MODE
LOG: (query) RESET statement_timeout
LOG: (query) SELECT pg_get_indexdef(indexrelid) FROM pg_index WHERE indrelid = $1 AND NOT indisvalid
LOG: 	(param:0) = 57452
LOG: (query) SELECT indexrelid, repack.repack_indexdef(indexrelid, indrelid, $2, FALSE)  FROM pg_index WHERE indrelid = $1 AND indisvalid
LOG: 	(param:0) = 57452
LOG: 	(param:1) = (null)
LOG: (query) SELECT repack.conflicted_triggers($1)
LOG: 	(param:0) = 57452
LOG: (query) CREATE TYPE repack.pk_57452 AS (c1 bigint)
LOG: (query) CREATE TABLE repack.log_57452 (id bigserial PRIMARY KEY, pk repack.pk_57452, row public.tmp_t0)
LOG: (query) CREATE TRIGGER repack_trigger AFTER INSERT OR DELETE OR UPDATE ON public.tmp_t0 FOR EACH ROW EXECUTE PROCEDURE repack.repack_trigger('INSERT INTO repack.log_57452(pk, row) VALUES( CASE WHEN $1 IS NULL THEN NULL ELSE (ROW($1.c1)::repack.pk_57452) END, $2)')
LOG: (query) ALTER TABLE public.tmp_t0 ENABLE ALWAYS TRIGGER repack_trigger
LOG: (query) SELECT repack.disable_autovacuum('repack.log_57452')
设置RC事务级别,查询除自身外对该表有AccessExclusiveLock 的进程号
LOG: (query) BEGIN ISOLATION LEVEL READ COMMITTED
LOG: (query) SELECT pg_backend_pid()
LOG: (query) SELECT pid FROM pg_locks WHERE locktype = 'relation' AND granted = false AND relation = 57452 AND mode = 'AccessExclusiveLock' AND pid <> pg_backend_pid()
LOG: (query) COMMIT
设置RS事务级别，串行操作，创建 repack.table_57462表结构，插入数据
LOG: (query) BEGIN ISOLATION LEVEL SERIALIZABLE
LOG: (query) SELECT set_config('work_mem', current_setting('maintenance_work_mem'), true)
LOG: (query) SET LOCAL synchronize_seqscans = off
LOG: (query) SELECT repack.array_accum(l.virtualtransaction)   FROM pg_locks AS l   LEFT JOIN pg_stat_activity AS a     ON l.pid = a.pid   LEFT JOIN pg_database AS d     ON a.datid = d.oid   WHERE l.locktype = 'virtualxid'   AND l.pid NOT IN (pg_backend_pid(), $1)   AND (l.virtualxid, l.virtualtransaction) <> ('1/1', '-1/0')   AND (a.application_name IS NULL OR a.application_name <> $2)  AND a.query !~* E'^\\s*vacuum\\s+'   AND a.query !~ E'^autovacuum: '   AND ((d.datname IS NULL OR d.datname = current_database()) OR l.database = 0)
LOG: 	(param:0) = 2132
LOG: 	(param:1) = pg_repack
LOG: (query) DELETE FROM repack.log_57452
LOG: (query) SELECT pid FROM pg_locks WHERE locktype = 'relation' AND granted = false AND relation = 57452 AND mode = 'AccessExclusiveLock' AND pid <> pg_backend_pid()
LOG: (query) SET LOCAL statement_timeout = 100
LOG: (query) LOCK TABLE public.tmp_t0 IN ACCESS SHARE MODE
LOG: (query) RESET statement_timeout
LOG: (query) CREATE TABLE repack.table_57452 WITH (oids = false) TABLESPACE pg_default AS SELECT c0,c1 FROM ONLY public.tmp_t0 WITH NO DATA
LOG: (query) INSERT INTO repack.table_57452 SELECT c0,c1 FROM ONLY public.tmp_t0
LOG: (query) SELECT repack.disable_autovacuum('repack.table_57452')
LOG: (query) COMMIT
LOG: (query) CREATE UNIQUE INDEX index_57455 ON repack.table_57452 USING btree (c1) TABLESPACE pg_default
创建索引
repack.table_57432 插入原始的基础的数据。
LOG: (query) SELECT repack.repack_apply($1, $2, $3, $4, $5, $6)
LOG: 	(param:0) = SELECT * FROM repack.log_57452 ORDER BY id LIMIT $1
LOG: 	(param:1) = INSERT INTO repack.table_57452 VALUES ($1.*)
LOG: 	(param:2) = DELETE FROM repack.table_57452 WHERE (c1) = ($1.c1)
LOG: 	(param:3) = UPDATE repack.table_57452 SET (c0, c1) = ($2.c0, $2.c1) WHERE (c1) = ($1.c1)
LOG: 	(param:4) = DELETE FROM repack.log_57452 WHERE id IN (
LOG: 	(param:5) = 1000
LOG: (query) SELECT pid FROM pg_locks WHERE locktype = 'virtualxid' AND pid <> pg_backend_pid() AND virtualtransaction = ANY($1)
LOG: 	(param:0) = {}
LOG: (query) SAVEPOINT repack_sp1
LOG: (query) SET LOCAL statement_timeout = 100
LOG: (query) LOCK TABLE public.tmp_t0 IN ACCESS EXCLUSIVE MODE
LOG: (query) RESET statement_timeout
LOG: (query) SELECT repack.repack_apply($1, $2, $3, $4, $5, $6)
LOG: 	(param:0) = SELECT * FROM repack.log_57452 ORDER BY id LIMIT $1
LOG: 	(param:1) = INSERT INTO repack.table_57452 VALUES ($1.*)
LOG: 	(param:2) = DELETE FROM repack.table_57452 WHERE (c1) = ($1.c1)
LOG: 	(param:3) = UPDATE repack.table_57452 SET (c0, c1) = ($2.c0, $2.c1) WHERE (c1) = ($1.c1)
LOG: 	(param:4) = DELETE FROM repack.log_57452 WHERE id IN (
LOG: 	(param:5) = 0
LOG: (query) SELECT repack.repack_swap($1)
LOG: 	(param:0) = 57452
LOG: (query) COMMIT
需要 LOCK TABLE public.tmp_t0 IN ACCESS EXCLUSIVE MODE
repack.table_57432插入变化的数据。
做切换 repack.repack_swap
LOG: (query) BEGIN ISOLATION LEVEL READ COMMITTED
LOG: (query) SAVEPOINT repack_sp1
LOG: (query) SET LOCAL statement_timeout = 100
LOG: (query) LOCK TABLE public.tmp_t0 IN ACCESS EXCLUSIVE MODE
LOG: (query) RESET statement_timeout
LOG: (query) SELECT repack.repack_drop($1, $2)
LOG: 	(param:0) = 57452
LOG: 	(param:1) = 4
LOG: (query) COMMIT
设置RC事务级别，处理旧表
LOG: (query) BEGIN ISOLATION LEVEL READ COMMITTED
LOG: (query) ANALYZE public.tmp_t0
LOG: (query) COMMIT
LOG: (query) SELECT pg_advisory_unlock($1, CAST(-2147483648 + $2::bigint AS integer))
LOG: 	(param:0) = 16185446
LOG: 	(param:1) = 57452
```

四、pg_repack原理

创建一个新表，将数据从旧表移动到新表。为了避免表被独占锁定，创建了一个额外的日志表来记录原始表的改动，还添加了一个把INSERT / UPDATE / DELETE操作记录到日志表的触发器。当原始表中的数据全部导入到新表中，索引重建完毕，日志表的改动全部完成，pg_repack会连同新索引，用新表替换旧表，并将原旧表Drop掉。整个过程非常简单，非常可靠，但是需要注意的是——需要额外剩余足够的磁盘空间（原表大小 + 索引 + 额外的日志表空间）。 

参看资料:

https://blog.csdn.net/ctypyb2002/article/details/85007105

https://blog.csdn.net/ctypyb2002/article/details/85043550

https://www.timbotetsu.com/blog/postgresql-bloatbusters/