

#                  odyssey性能测试

本文档主要测试odyssey的性能，采用transaction模式，测试使用pgbench的simple协议。

测试方法是对比不同数量级别的使用odyssey和未使用odyssey的数据库性能。

## 1.环境

| IP           | 系统     | 中间件  | postgresql | 内存 | cpu    | 硬盘        |
| ------------ | -------- | ------- | ---------- | ---- | ------ | ----------- |
| 192.168.6.15 | rhel 7.5 | odyssey | 11.1       | 256G | 2x28核 | 2x480GB SSD |

## 2.测试环境准备

#### 2.1 odyssey中间件安装

参考：《Odyssey编译安装配置文档》

#### 2.2 创建用于测试的数据库

```
create database pgbench;
```

#### 2.3 配置测试数据库连接的odyssey登录配置文件

```
storage "tpcb" {
        type "remote"
        host "127.0.0.1"
        port 8432
}
database "pgbench" {
         user "postgres" {
              authentication "none"
             password "postgres"
             client_max 100
             storage "tpcb"
             storage_db "pgbench"
             storage_user "postgres"
             storage_password "postgres"
             pool "transaction"
             pool_size 112
             pool_timeout 10
             pool_ttl 1000
             pool_cancel yes
             pool_rollback yes
             client_fwd_error no
             log_debug no
             auth_common_name "postgres"
         }
}
```

## 3.测试

#### 3.1 测试1（100W数据）

##### 3.11 初始化

```
pgbench -i -F 100 -s 10 pgbench
```

##### 3.12 只读测试语句（select_only）

使用odyssey中间件登录：

```
pgbench -M simple -p 7432 -U postgres -h 127.0.0.1 -P 1 -n -r -c 56 -C pgbench -T 300 -S
```

未使用odyssey中间件登录：

```
pgbench -M simple -p 8432 -U postgres -h 127.0.0.1 -P 1 -n -r -c 56 -C pgbench -T 300 -S
```

##### 3.13 TPCB测试语句

使用odyssey中间件登录：

```
pgbench -M simple -p 7432 -U postgres -h 127.0.0.1 -P 1 -n -r -c 56 -C pgbench -T 300
```

未使用odyssey中间件登录：

```
pgbench -M simple -p 8432 -U postgres -h 127.0.0.1 -P 1 -n -r -c 56 -C pgbench -T 300
```

##### 3.14 测试结果

（1）只读测试结果

使用odyssey中间件登录：

```
transaction type: <builtin: select only>
scaling factor: 10
query mode: simple
number of clients: 56
number of threads: 1
duration: 300 s
number of transactions actually processed: 318338
latency average = 51.163 ms
latency stddev = 175.511 ms
tps = 1061.121141 (including connections establishing)
tps = 1079.773269 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
        51.162  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```

未使用odyssey中间件登录：

```
transaction type: <builtin: select only>
scaling factor: 10
query mode: simple
number of clients: 56
number of threads: 1
duration: 300 s
number of transactions actually processed: 120681
latency average = 136.713 ms
latency stddev = 18.344 ms
tps = 402.264776 (including connections establishing)
tps = 409.463883 (excluding connections establishing)
statement latencies in milliseconds:
         0.002  \set aid random(1, 100000 * :scale)
       136.711  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```



（2）TPCB测试结果

使用odyssey中间件登录：

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 56
number of threads: 1
duration: 300 s
number of transactions actually processed: 293821
latency average = 56.269 ms
latency stddev = 174.590 ms
tps = 978.950302 (including connections establishing)
tps = 994.914785 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
         0.000  \set bid random(1, 1 * :scale)
         0.000  \set tid random(1, 10 * :scale)
         0.000  \set delta random(-5000, 5000)
         3.985  BEGIN;
         4.078  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         4.133  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
        14.299  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
        21.817  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         4.116  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         3.837  END;
```

未使用odyssey中间件登录：

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 56
number of threads: 1
duration: 300 s
number of transactions actually processed: 117561
latency average = 140.420 ms
latency stddev = 88.530 ms
tps = 391.853169 (including connections establishing)
tps = 398.645003 (excluding connections establishing)
statement latencies in milliseconds:
         0.002  \set aid random(1, 100000 * :scale)
         0.001  \set bid random(1, 1 * :scale)
         0.000  \set tid random(1, 10 * :scale)
         0.000  \set delta random(-5000, 5000)
         9.729  BEGIN;
        10.393  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
        10.203  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
        35.511  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
        54.823  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
        10.314  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         9.444  END;
```

#### 3.2 测试2 （1000W数据）

##### 3.2.1 初始化

```
pgbench -i -F 100 -s 100 pgbench
```

##### 3.2.2 测试语句

测试语句同测试1。

##### 3.2.3 测试结果

（1）只读测试结果

使用odyssey中间件登录：

```
transaction type: <builtin: select only>
scaling factor: 100
query mode: simple
number of clients: 56
number of threads: 1
duration: 300 s
number of transactions actually processed: 1267987
latency average = 13.029 ms
latency stddev = 2.815 ms
tps = 4226.606201 (including connections establishing)
tps = 4292.474580 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
        13.028  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```

未使用odyssey中间件登录：

```
transaction type: <builtin: select only>
scaling factor: 100
query mode: simple
number of clients: 56
number of threads: 1
duration: 300 s
number of transactions actually processed: 120248
latency average = 137.206 ms
latency stddev = 18.413 ms
tps = 400.815685 (including connections establishing)
tps = 407.988381 (excluding connections establishing)
statement latencies in milliseconds:
         0.002  \set aid random(1, 100000 * :scale)
       137.204  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```



（2）TPCB测试结果

使用odyssey中间件登录：

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 56
number of threads: 1
duration: 300 s
number of transactions actually processed: 879637
latency average = 18.848 ms
latency stddev = 4.875 ms
tps = 2932.084930 (including connections establishing)
tps = 2968.350306 (excluding connections establishing)
statement latencies in milliseconds:
         0.002  \set aid random(1, 100000 * :scale)
         0.000  \set bid random(1, 1 * :scale)
         0.000  \set tid random(1, 10 * :scale)
         0.000  \set delta random(-5000, 5000)
         2.575  BEGIN;
         2.634  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         2.612  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         2.720  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
         3.119  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         2.586  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         2.600  END;
```

未使用odyssey中间件登录：

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 56
number of threads: 1
duration: 300 s
number of transactions actually processed: 117051
latency average = 141.028 ms
latency stddev = 32.180 ms
tps = 390.145469 (including connections establishing)
tps = 396.913979 (excluding connections establishing)
statement latencies in milliseconds:
         0.002  \set aid random(1, 100000 * :scale)
         0.000  \set bid random(1, 1 * :scale)
         0.000  \set tid random(1, 10 * :scale)
         0.000  \set delta random(-5000, 5000)
        18.985  BEGIN;
        19.876  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
        19.504  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
        20.389  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
        23.490  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
        19.455  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
        19.324  END;
```

#### 3.3 测试3 （1亿数据）

##### 3.31 初始化

```
pgbench -i -F 100 -s 1000 pgbench
```

##### 3.32 测试语句

测试语句同测试1。

##### 3.3.3 测试结果

（1）只读测试结果

使用odyssey中间件登录：

```
transaction type: <builtin: select only>
scaling factor: 1000
query mode: simple
number of clients: 56
number of threads: 1
duration: 300 s
number of transactions actually processed: 1245607
latency average = 13.264 ms
latency stddev = 2.760 ms
tps = 4152.004636 (including connections establishing)
tps = 4216.713130 (excluding connections establishing)
statement latencies in milliseconds:
         0.001  \set aid random(1, 100000 * :scale)
        13.262  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```

未使用odyssey中间件登录：

```
transaction type: <builtin: select only>
scaling factor: 1000
query mode: simple
number of clients: 56
number of threads: 1
duration: 300 s
number of transactions actually processed: 119469
latency average = 138.100 ms
latency stddev = 18.532 ms
tps = 398.201110 (including connections establishing)
tps = 405.328277 (excluding connections establishing)
statement latencies in milliseconds:
         0.002  \set aid random(1, 100000 * :scale)
       138.098  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```

（2）TPCB测试结果

使用odyssey中间件登录：

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1000
query mode: simple
number of clients: 56
number of threads: 1
duration: 300 s
number of transactions actually processed: 830586
latency average = 19.962 ms
latency stddev = 4.943 ms
tps = 2768.591629 (including connections establishing)
tps = 2802.695593 (excluding connections establishing)
statement latencies in milliseconds:
         0.002  \set aid random(1, 100000 * :scale)
         0.001  \set bid random(1, 1 * :scale)
         0.000  \set tid random(1, 10 * :scale)
         0.000  \set delta random(-5000, 5000)
         2.798  BEGIN;
         2.950  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         2.836  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         2.838  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
         2.892  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         2.802  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         2.843  END;
```

未使用odyssey中间件登录：

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1000
query mode: simple
number of clients: 56
number of threads: 1
duration: 300 s
number of transactions actually processed: 102529
latency average = 160.987 ms
latency stddev = 34.850 ms
tps = 341.736150 (including connections establishing)
tps = 347.696476 (excluding connections establishing)
statement latencies in milliseconds:
         0.002  \set aid random(1, 100000 * :scale)
         0.001  \set bid random(1, 1 * :scale)
         0.000  \set tid random(1, 10 * :scale)
         0.000  \set delta random(-5000, 5000)
        22.648  BEGIN;
        23.433  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
        22.570  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
        22.931  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
        23.344  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
        22.735  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
        23.324  END;
```

## 4. 测试结果汇总

说明：

表格中Y 代表使用odyssey中间件，N 代表不使用odyssey中间件

#### 4.1 只读测试

| 数据库         | 数据量 | odyssey | QPS  | latency average(ms) | latency stddev(ms) |
| -------------- | ------ | ------- | ---- | ------------------- | ------------------ |
| postgres 11.1  | 100W   | Y       | 1080 | 51.163              | 175.511            |
| postgres  11.1 | 100W   | N       | 410  | 203.345             | 48.497             |
| postgres 11.1  | 1000W  | Y       | 4293 | 13.029              | 2.815              |
| postgres 11.1  | 1000W  | N       | 408  | 137.206             | 18.413             |
| postgres 11.1  | 1亿    | Y       | 4217 | 13.264              | 2.760              |
| postgres 11.1  | 1亿    | N       | 405  | 138.100             | 18.532             |

#### 4.2 TPCB测试

| 数据库        | 数据量 | odyssey | TPS  | latency average(ms) | latency stddev(ms) |
| ------------- | ------ | ------- | ---- | ------------------- | ------------------ |
| postgres 11.1 | 100W   | Y       | 995  | 56.269              | 174.590            |
| postgres 11.1 | 100W   | N       | 399  | 140.420             | 88.530             |
| postgres 11.1 | 1000W  | Y       | 2969 | 18.848              | 4.875              |
| postgres 11.1 | 1000W  | N       | 397  | 141.028             | 32.180             |
| postgres 11.1 | 1亿    | Y       | 2803 | 19.962              | 4.943              |
| postgres 11.1 | 1亿    | N       | 348  | 160.987             | 34.850             |

从上面测试结果来看：

（1）使用odyssey中间件连接数据库的只读性能是直连数据库的10倍左右(1000W数据量以上)。

（2）使用odyssey中间件连接数据库的tpcb整体性能是直连数据库的7.5~8.0倍左右(1000W数据量以上)。

（3）不论是只读还是tpcb在使用了odyssey中间件连接数据库后比直连数据库的平均等待时间要大幅度降低，而且稳定性也比直连数据库要好(1000W数据量以上)。

