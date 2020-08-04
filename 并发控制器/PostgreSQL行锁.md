# 			PostgreSQL行锁实验

### 一、FOR UPDATE模式

```sql
-- 这种行锁模式会将SELECT读取到的行进行"更新"锁定，不允许其他事务UPDATE、DELETE、SELECT FOR UPDATE、SELECT FOR NO KEY UPDATE、SELECT FOR SHARE或者SELECT FOR KEY SHARE这些行的事务将被阻塞。

-- 获取锁模式的途径:SELECT FOR UPDATE、DELETE、UPDATE(更新的是唯一索引那一列)

-- 实际获得:排他锁

-- session1
test=# start transaction; 
START TRANSACTION
test=# select txid_current();
 txid_current 
--------------
          932
(1 row)

-- session2
test=# start transaction;
START TRANSACTION
test=# select txid_current();
 txid_current 
--------------
          933
(1 row)

-- session1获取FOR UPDATE锁
test=# select * from test for update;
 id | name 
----+------
  1 | edw
  2 | edw
  4 | edw
  6 | edw
  7 | edw
  8 | edw
  9 | edw
 10 | edw
(8 rows)

-- session2尝试"更新模式"，发现更新被锁定
-- SELECT FOR KEY SHARE也会被锁定
test=# delete from test where id=2;


-- session1 commit释放FOR UPDATE锁
test=# delete from test where id=1;
DELETE 1
test=# select * from test;
 id | name 
----+------
  2 | edw
  4 | edw
  6 | edw
  7 | edw
  8 | edw
  9 | edw
 10 | edw
(7 rows)

test=# commit;
COMMIT

-- session2获取FOR UPDATE锁
test=# delete from test where id=2;
DELETE 1
test=# select * from test;
 id | name 
----+------
  4 | edw
  6 | edw
  7 | edw
  8 | edw
  9 | edw
 10 | edw
(6 rows)

test=# commit
```

### 二、FOR NO KEY UPDATE 

```sql
-- 和UPDATE类似，也是进行"更新"锁定,但是比FOR UPDATE弱一点，不锁SELECT FOR KEY SHARE

-- 获取锁模式的途径:UPDATE(除了更新索引列之外的UPDATE)

-- 实际获得:排他锁

-- session1 获取FOR NO KEY UPDATE锁
test=# start transaction;
START TRANSACTION
test=# update test set name='test_for_update' where id<10;
UPDATE 4

-- session2 尝试SELECT FOR KEY SHARE,发现没有锁定
test=# start transaction;
START TRANSACTION
test=# select * from test where id<10 for key share;
 id |    name    
----+------------
  6 | edw
  8 | edw
  9 | edw
  7 | test_share
(4 rows)

-- session2 尝试更新被锁定
test=# update test set name='test' where id=8;
UPDATE 1

-- session1 commit，释放FOR NO KEY UPDATE锁
test=# commit;
COMMIT

-- session2 解锁，获取FOR UPDATE锁
test=# update test set name='test' where id=8;
UPDATE 1
test=# select * from test;
 id |      name       
----+-----------------
 10 | edw
  6 | test_for_update
  9 | test_for_update
  7 | test_for_update
  8 | test
(5 rows)
test=# commit;
COMMIT
```
### 三、FOR SHARE

```sql
-- 和FOR UPDATE类似，但是获取的是共享锁，一个共享锁会阻塞其他事务在这些行上执行UPDATE、DELETE、SELECT FOR UPDATE或者SELECT FOR NO KEY UPDATE，但是它不会阻止它们执行SELECT FOR SHARE或者SELECT FOR KEY SHARE。

-- 获取锁模式的途径:SELECT FOR SHARE

-- 实际获得:共享锁

-- session1 获取FOR SHARE锁
test=# start transaction;
START TRANSACTION
test=# select * from test for share;
 id |      name       
----+-----------------
 10 | edw
  6 | test_for_update
  9 | test_for_update
  7 | test_for_update
  8 | test
(5 rows)

-- session2 尝试FOR KEY SHARE锁，没有发生锁定
test=# start transaction;
START TRANSACTION
test=# select * from test where id=8 for key share;
 id | name 
----+------
  8 | test
(1 row)

--session2 尝试更新发生锁定
test=# update test set name='test_share' where id=8;

-- session1 commit,释放FOR SHARE锁
test=# commit;
COMMIT

-- session2 解锁，获得FOR UPDATE锁
test=# update test set name='test_share' where id=8;
UPDATE 1
test=# select * from test;
 id |      name       
----+-----------------
 10 | edw
  6 | test_for_update
  9 | test_for_update
  7 | test_for_update
  8 | test_share
(5 rows)
test=# commit;
COMMIT
```

### 四、FOR KEY SHARE

```sql
-- 和FOR SHARE类似，也是锁更新，也是共享锁,但是比FOR SHARE弱一点，不锁定SELECT FOR NO KEY UPDATE 

-- 获取锁模式的途径:SELECT FOR KEY SHARE

-- 实际获得:共享锁

-- session1 获取FOR KEY SHARE锁
test=# start transaction;
START TRANSACTION
test=# select * from test for key share;
 id |      name       
----+-----------------
 10 | edw
  6 | test_for_update
  9 | test_for_update
  7 | test_for_update
  8 | test_share
(5 rows)

-- session2 尝试SELECT FOR NO KEY UPDATE,发现没有锁定
test=# start transaction;
START TRANSACTION
test=# update test set name='test_key' where id=8;
UPDATE 1

-- session2 尝试FOR UPDATE锁，锁住
test=# select * from test for update;

-- session1 commit,释放FOR KEY SHARE
test=# commit;
COMMIT

-- session2
test=# select * from test for update;
 id |      name       
----+-----------------
 10 | edw
  6 | test_for_update
  9 | test_for_update
  7 | test_for_update
  8 | test_key
(5 rows)
test=# commit;
COMMIT
```

### 五、行锁的实现机制

```sql
-- 主要借助事务锁，并且将版本的更新状态到t_xmax和t_infomask中，其他更新操作之前根据可见性检查规则，检查当前t_xmax的状态(t_informask中)，如果t_xmax正在进行，其他会话等待行锁(等待事务锁)看到的是旧版本的数据；多个等待事务，按请求的先后顺序分配行锁，得到行锁之后又按照规则进行锁检查。
-- 参考连接
http://mysql.taobao.org/monthly/2018/12/07/
```

### 总结

​	PostgreSQL这四种锁模式其实就是排他锁和共享锁在一些特殊情况下的具体分类(为了FOR NO KEY UPDATE和FOR  KEY SHARE不冲突，将原来的两种变为四种模式)。排他锁，会锁住其他并发事务的更新操作，但是不影响查询；共享锁，共享锁之间可以共存，但是有了共享锁就会和排他锁冲突。