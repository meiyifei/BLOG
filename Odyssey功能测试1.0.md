# 				Odyssey功能测试

## Odyssey设计的目标和主要特性介绍

### 1、多线程处理

通过指定一些额外的工作线程，Odyssey可以显著提高处理性能。每个工作线程负责身份验证和代理客户机到服务器和服务器到客户机的请求。所有工作线程都共享全局服务器连接池。多线程设计在SSL/TLS性能中起着重要的作用。 

### 2、高级事务池

Odyssey跟踪当前事务状态，如果发生意外的客户端断开连接，可以发出自动取消连接并回滚已放弃的事务，然后将服务器连接放回服务器池以便重用。此外，记住最后的服务器连接属于客户端，以减少在每个客户机到服务器的分配上设置客户机选项的需要。

### 3、更好的连接池控制 

Odyssey允许将连接池定义为一对数据库和用户。每个定义的池可以有单独的身份验证、池模式和限制设置。 

### 4、身份认证

Odyssey提供了功能齐全的SSL/TLS支持，并提供了常见的身份验证方法，例如:md5和clear text，用于客户机和服务器身份验证。此外，它允许单独阻塞每个池用户。 

### 5、日志记录

Odyssey为客户机和服务器连接生成通用惟一标识符uuid。任何日志事件和客户机错误响应都包含id，然后可以使用id惟一地标识客户机和跟踪操作。Odyssey可以使用系统日志记录器将日志事件保存到日志文件中。 



## Odyssey相比与PgBouncer的优势

​	pgbouncer是一个轻量级的连接池，效率高。主要适用的场景是高并发、短链接的情况下，对于长事务用户可以单独配置其连接池。

​	但是由于pgbouncer是单进程模式，导致一个pgbouncer服务本身最多只能使用1核 。要解决这个弊端，可以使用多个pgbouncer服务，但是管理起来比较麻烦。

​	而Odyssey则是多线程模式，可以指定多个工作线程，充分利用cpu资源，显著提高处理性能 。

这也可以解决pgbouncer单进程的瓶颈。

​	



## 1、连接池模型测试

### 测试说明: 

​	目前Odyssey支持的连接池模式有"session"和“transaction”两种模式

### 测试初始化:  

​	将odyssey的配置文件中pool参数分别设置为“session”和“transaction”、“statement”

### 测试过程:

```
(1)“session”模式

1、通过Odyssey连接数据库(session 1)
psql -p6432 -h127.0.0.1 -U postgres -d postgres

日志信息:
3411 12 Mar 14:03:38.077 info [none none] (stats) [postgres.postgres] 1 clients, 1 active servers, 0 idle servers, 0 transactions/sec (0 usec) 0 queries/sec (0 usec) 0 in bytes/sec, 0 out bytes/sec

2、再开一个会话通过Odyssey连接到数据库(session 2)
日志信息：
3411 12 Mar 14:09:40.232 info [none none] (stats) [postgres.postgres] 2 clients, 2 active servers, 0 idle servers, 0 transactions/sec (0 usec) 0 queries/sec (0 usec) 0 in bytes/sec, 0 out bytes/sec

3、依次断开session1和session2
日志信息:
3411 12 Mar 14:11:48.481 info [c7dd065230b55 s7ad5cf188741] (main) client disconnected
3411 12 Mar 14:11:53.589 info [ce436792dca2c s4dc7f17ac34c] (main) client disconnected
3411 12 Mar 14:14:54.599 info [none none] (stats) worker[0]: msg (2 allocated, 0 cached, 1 freed, 0 cache_size), coroutines (5 active, 0 cached), clients_processed: 0


(2)“transaction模式”
1、通过Odyssey连接数据库
psql -p6432 -h127.0.0.1 -U postgres -d postgres

日志信息:
3668 12 Mar 14:27:33.401 info [none none] (stats) clients 1
3668 12 Mar 14:27:33.401 info [none none] (stats) [postgres.postgres] 1 clients, 0 active servers, 1 idle servers, 0 transactions/sec (15471 usec) 0 queries/sec (15471 usec) 5 in bytes/sec, 8 out bytes/sec


2、开启事务
配置文件的配置:
storage "postgres_server" {
	type "remote"
	host "127.0.0.1"
	port 1922
}

database "pgbench" {
	user "postgres" {
		authentication "none"
		storage "postgres_server"
		storage_db "pgbench"
		storage_user "postgres"
		storage_password "123456"
		pool "transaction"
		pool_size 112
		pool_timeout 0
		pool_ttl 60
		pool_discard no
		pool_cancel yes
		pool_rollback yes
		client_fwd_error yes
		log_debug no
	}
}
初始化:
pgbench -i -F 100 -s 10 pgbench
使用Odyssey连接数据库，进行批量的业务处理(处理20个事务)
pgbench  -p 6432 -U postgres -h 127.0.0.1 -P 1 -n -r  -C pgbench -t 20  -S
部分日志信息:
4147 12 Mar 15:34:07.324 info [c790be0499738 none] (startup) new client connection 127.0.0.1:38261
4147 12 Mar 15:34:07.324 info [c790be0499738 none] (startup) route 'pgbench.postgres' to 'pgbench.postgres'
4147 12 Mar 15:34:07.324 info [c790be0499738 none] (setup) login time: 97 microseconds
4147 12 Mar 15:34:07.324 info [c790be0499738 s42f5fb7d7c0e] (main) client disconnected
4147 12 Mar 15:34:07.324 info [c4e66e45c1eb7 none] (startup) new client connection 127.0.0.1:38262
4147 12 Mar 15:34:07.324 info [c4e66e45c1eb7 none] (startup) route 'pgbench.postgres' to 'pgbench.postgres'
4147 12 Mar 15:34:07.324 info [c4e66e45c1eb7 none] (setup) login time: 82 microseconds
4147 12 Mar 15:34:07.325 info [c4e66e45c1eb7 s42f5fb7d7c0e] (main) client disconnected
4147 12 Mar 15:34:07.327 info [c04db5d6c104e s42f5fb7d7c0e] (main) client disconnected

发现确实是为每个事务分配一个数据库连接，事务执行完毕释放连接。


(3)“statement”模式:
[root@Odyssey sources]# ./odyssey /usr/local/src/odyssey/odyssey.conf 
4181 12 Mar 15:39:38.972 error (rules) rule 'pgbench.postgres': unknown pooling mode

发现启动时报错，Odyssey没有pgbouncer的“statement”模式

```

### 预期测试结果:

​	“session”模式下，在它的生命周期内，连接池分配给他一个数据库连接。客户端断开时，数据库连接会放回连接池中。

​	“transaction”模式下，当客户端的每个事务结束时，数据库连接就会重新释放回连接池中，再次执行一个事务时，需要再从连接池获得一个连接。

​	“statement”模式下,启动Odyssey时报错。

### 测试结果：

​	“session”模式下，每个客户端连接数据库时就分配一个连接，客户端断开就释放连接。

​	”transaction“模式下，为每个事务建立一个连接，事务结束时释放连接。

​	”statement“模式下，启动时报错。

### 测试结论：

​	 Odyssey支持”session“和”transaction”模式，不支持“statement”模式。



## 2、最大客户端连接数

### 测试说明:

​	Odyssey中设置每个路由的客户端连接限制的参数为client_max,注释此项表示禁用该限制。

​	这个参数分为全局和每个路由，全局的大于每个路由的。

### 测试初始化：

```
主要配置文件如下:

storage "postgres_server" {
       type "remote"
        host "127.0.0.1"
        port 1922
}

database "pgbench" {
        user "postgres" {
                authentication "none"
                client_max 4
                storage "postgres_server"
                storage_db "pgbench"
                storage_user "postgres"
                storage_password "123456"
                pool "session"
                pool_size 2
                pool_timeout 6000
                pool_ttl 60
                pool_discard no
                pool_cancel yes
                pool_rollback yes
                log_debug no
        }
}

storage "postgres_server1" {
        type "local"
        host "127.0.0.1"
        port 1922
}


database "postgres" {
        user "postgres" {
                authentication "md5"
               password "111111"
                storage "postgres_server1"
                storage_db "postgres"
                storage_user "postgres"
                storage_password "123456"
                pool "transaction"
                pool_size 0
                pool_timeout 0
                pool_discard no
                pool_cancel yes
                log_debug no
        }
}
```

### 测试过程:

```
在"session"或者“transaction”模式下
可以看到不论是那个模式,都是只连接到Odyssey上,还没有分配数据库连接处于等待状态。

postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 51533 | 127.0.0.1  |       6432 |              |              | cd34ab24b260d |      |          0 | 
 C    | postgres | pgbench  | pending | 127.0.0.1 | 51539 | 127.0.0.1  |       6432 |              |              | c9c8fec491937 |      |          0 | 
 C    | postgres | pgbench  | pending | 127.0.0.1 | 51540 | 127.0.0.1  |       6432 |              |              | c63706277eb9b |      |          0 | 
 C    | postgres | pgbench  | pending | 127.0.0.1 | 51541 | 127.0.0.1  |       6432 |              |              | c2400d0170eff |      |          0 | 
 C    | postgres | pgbench  | pending | 127.0.0.1 | 51542 | 127.0.0.1  |       6432 |              |              | cd16be54dea16 |      |          0 | 
(5 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         0 |          4 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | session
        | c2400d0170eff |      |          0 | 


再次连接到Odyssey上报错
[postgres@Odyssey ~]$ psql -p 6432 -h 127.0.0.1 -U postgres -d pgbench
psql: server closed the connection unexpectedly
	This probably means the server terminated abnormally
	before or while processing the request.


[postgres@Odyssey ~]$ ps -ef | grep odyssey
postgres  4980  4940  0 15:43 pts/6    00:00:00 grep odyssey

此时Odyssey进程已经关闭
```

### 预期测试结果：

​	当客户端通过Odyssey连接数据库时，如果客户端数目超过4个，则会出现“too many connections”的错误。

### 测试结果：

​	当第5个客户端连接上来，Odyssey直接报错，此时Odyssey进程已经关闭。

### 测试结论:

​	client_max参数可以限制连接到Odyssey上客户端的最大数量。

## 3、连接超时时间和连接池连接数测试

### 测试说明:

​	Odyssey中控制连接池连接数和超时时间的是pool_size和pool_timeout这两个参数

### 测试初始化：

​	分别在“session”模式下和“transaction”模式下测试，主要配置文件如下

```
storage "postgres_server" {
       type "remote"
        host "127.0.0.1"
        port 1922
}

database "pgbench" {
        user "postgres" {
                authentication "none"
                client_max 4
                storage "postgres_server"
                storage_db "pgbench"
                storage_user "postgres"
                storage_password "123456"
                pool "session"
                pool_size 2
                pool_timeout 6000
                pool_ttl 60
                pool_discard no
                pool_cancel yes
                pool_rollback yes
                log_debug no
        }
}

storage "postgres_server1" {
        type "local"
        host "127.0.0.1"
        port 1922
}


database "postgres" {
        user "postgres" {
                authentication "md5"
               password "111111"
                storage "postgres_server1"
                storage_db "postgres"
                storage_user "postgres"
                storage_password "123456"
                pool "transaction"
                pool_size 0
                pool_timeout 0
                pool_discard no
                pool_cancel yes
                log_debug no
        }
}
```



### 测试过程：

```
在4个客户端中，先开启一个事务

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         1 |          3 |         1 |       0 |       0 |         0 |        0 |       0 |          0 | session
(2 rows)

postgres=> show clents;
ERROR:  odyssey: ce5796b14a281: console command error: show clents;
postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 51544 | 127.0.0.1  |       6432 |              |              | ce5796b14a281 |      |          0 | 
 C    | postgres | pgbench  | active  | 127.0.0.1 | 51545 | 127.0.0.1  |       6432 |              |              | c0ecc21310ea2 |      |          0 | 
 C    | postgres | pgbench  | pending | 127.0.0.1 | 51547 | 127.0.0.1  |       6432 |              |              | c2542db448430 |      |          0 | 
 C    | postgres | pgbench  | pending | 127.0.0.1 | 51548 | 127.0.0.1  |       6432 |              |              | ca2a66a014589 |      |          0 | 
 C    | postgres | pgbench  | pending | 127.0.0.1 | 51549 | 127.0.0.1  |       6432 |              |              | c3b19003be9f3 |      |          0 | 

在次开启一个事务(这才是真正的占住了这个连接)
postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         2 |          2 |         2 |       0 |       0 |         0 |        0 |       0 |          0 | session

再次开启一个事务，等待超时时间后报错。
pgbench=# start transaction;
ERROR:  odyssey: cc51e3f2ab359: failed to get remote server connection
server closed the connection unexpectedly
	This probably means the server terminated abnormally
	before or while processing the request.
The connection to the server was lost. Attempting reset: Succeeded.


下面看一下Odyssey是如何处理等待队列的
无论在session模式下还是transaction模式下只有开启事务才算真正的占住了这个连接。

一、在没有释放连接的情况下，客户端等待队列中请求失败后断开连接重新放入客户端等待队列中
二、在session模式下，在有连接释放的情况下看Odyssey如何处理请求队列的？(此时pool_size已满)
	在session模式下，开启事务才算真正的占住了这个连接，但是commit或者rollback这个事务，这个连接依然不释放。


postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         2 |          2 |         2 |       0 |       0 |         0 |        0 |       0 |          0 | session
(2 rows)


假设此时释放客户端连接，同时先后在客户端请求队列中start transaction(申请这个服务器连接)


postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         2 |          1 |         2 |       0 |       0 |         0 |        0 |       0 |          0 | session
(2 rows)

结论:可以看到先先进入请求队列的得到数据库连接，后面的请求失败(先进先出，而与进入客户端等待队列无关)，等待超时时间后重新进入客户端等待队列。
(这与PgBouncer不同，在PgBouncer中连接到PgBouncer后是不出现在客户端等待队列中，只有请求服务器连接失败后才会进入客户端等待队列，也就是说在PgBouncer中客户端等待队列的顺序和请求队列的顺序一致)


三、在transaction模式下，在有连接释放的情况下看Odyssey如何处理请求队列的？(此时pool_size已满)
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         2 |          2 |         2 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
(2 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         2 |          0 |         2 |       0 |       0 |         0 |        0 |       0 |          0 | transaction


上面的变化反映的是，客户端等待队列到服务器请求队列的变化

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         2 |          1 |         2 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
(2 rows)

在这里我们是提交了一个事务，也就是释放了一个连接(在transaction模式下)，可以看到服务器连接释放后进入客户端等待队列，请求队列依然是先进先出。

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         2 |          2 |         2 |       0 |       0 |         0 |        0 |       0 |          0 | transaction

后面的连接没有得到服务器连接，重新回到客户端等待队列。
```

### 预期测试结果:

​	连接数超过pool_size指定的数目，会进入一个请求等待队列中。如果请求队列的客户端超过pool_timeout则会报错重新进入客户端等待队列中。

### 测试结果:

​	连接个数超过pool_size时，等待一个超时时间，在超时时间内连接到服务器连接则分配给这个客户端，否则报错并进入等待队列中。

### 测试结论:

​	在“session”和“transaction”模式下，连接超时时间和连接数测试通过。

## 4、连接空闲超时时间测试

### 测试说明:

​	Odyssey控制连接空闲超时时间的参数时pool_ttl,设置为0表示禁用。(与连接复用有关)

### 测试初始化:

​	为了便于测试将这个值设置为60秒，分别在“session”和“transaction”模式下测试。

### 测试过程：

```
"transaction"模式：

（1）初始状态:

postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | pgbench  | pending | 127.0.0.1 | 55400 | 127.0.0.1  |       6432 |              |              | c0fc6e4474d38 |      |          0 | 
 C    | postgres | postgres | pending | 127.0.0.1 | 55403 | 127.0.0.1  |       6432 |              |              | c02e9da18d33d |      |          0 | 
(2 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 pgbench  | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transactio(2 rows)

postgres=> show servers;
 type | user | database | state | addr | port | local_addr | local_port | connect_time | request_time | ptr | link | remote_pid | tls 
------+------+----------+-------+------+------+------------+------------+--------------+--------------+-----+------+------------+-----
(0 rows)

(2)连接到数据库服务器(start transaction)

postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | pgbench  | active  | 127.0.0.1 | 55400 | 127.0.0.1  |       6432 |              |              | c0fc6e4474d38 |      |          0 | 
 C    | postgres | postgres | pending | 127.0.0.1 | 55403 | 127.0.0.1  |       6432 |              |              | c02e9da18d33d |      |          0 | 
(2 rows)

postgres=> show servers;
 type |   user   | database | state  |   addr    | port | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+--------+-----------+------+------------+------------+--------------+--------------+---------------+------+------------+-----
 S    | postgres | pgbench  | active | 127.0.0.1 | 1922 | 127.0.0.1  |      51467 |              |              | s7bbeda42e119 |      |          0 | 
(1 row)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 pgbench  | postgres |         1 |          0 |         1 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
(2 rows)

(3)提交这个事务

postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | pgbench  | pending | 127.0.0.1 | 55400 | 127.0.0.1  |       6432 |              |              | c0fc6e4474d38 |      |          0 | 
 C    | postgres | postgres | pending | 127.0.0.1 | 55403 | 127.0.0.1  |       6432 |              |              | c02e9da18d33d |      |          0 | 
(2 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 pgbench  | postgres |         0 |          1 |         0 |       1 |       0 |         0 |        0 |       0 |          0 | transaction
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
(2 rows)

postgres=> show servers;
 type |   user   | database | state |   addr    | port | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+-------+-----------+------+------------+------------+--------------+--------------+---------------+------+------------+-----
 S    | postgres | pgbench  | idle  | 127.0.0.1 | 1922 | 127.0.0.1  |      51467 |              |              | s7bbeda42e119 |      |          0 | 
(1 row)

提交事务后服务器连接并不是立即回收，而是处于空闲的状态。

(4)等待pool_ttl时间后，服务器连接断开

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 pgbench  | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
(2 rows)

postgres=> show servers;
 type | user | database | state | addr | port | local_addr | local_port | connect_time | request_time | ptr | link | remote_pid | tls 
------+------+----------+-------+------+------+------------+------------+--------------+--------------+-----+------+------------+-----
(0 rows)
```

```
"session"模式下测试:

（1）初始状态

postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | pgbench  | pending | 127.0.0.1 | 55406 | 127.0.0.1  |       6432 |              |              | c25270604e2c5 |      |          0 | 
 C    | postgres | postgres | pending | 127.0.0.1 | 55409 | 127.0.0.1  |       6432 |              |              | c65859260c452 |      |          0 | 
(2 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 pgbench  | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | session
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
(2 rows)

postgres=> show servers;
 type | user | database | state | addr | port | local_addr | local_port | connect_time | request_time | ptr | link | remote_pid | tls 
------+------+----------+-------+------+------+------------+------------+--------------+--------------+-----+------+------------+-----
(0 rows)


	（2）连接到数据库服务器(start transaction)
	postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | pgbench  | active  | 127.0.0.1 | 55406 | 127.0.0.1  |       6432 |              |              | c25270604e2c5 |      |          0 | 
 C    | postgres | postgres | pending | 127.0.0.1 | 55409 | 127.0.0.1  |       6432 |              |              | c65859260c452 |      |          0 | 
(2 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 pgbench  | postgres |         1 |          0 |         1 |       0 |       0 |         0 |        0 |       0 |          0 | session
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
(2 rows)

postgres=> show servers;
 type |   user   | database | state  |   addr    | port | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+--------+-----------+------+------------+------------+--------------+--------------+---------------+------+------------+-----
 S    | postgres | pgbench  | active | 127.0.0.1 | 1922 | 127.0.0.1  |      51473 |              |              | se2bb35408515 |      |          0 | 
(1 row)

（3）提交这个事务

postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | pgbench  | active  | 127.0.0.1 | 55406 | 127.0.0.1  |       6432 |              |              | c25270604e2c5 |      |          0 | 
 C    | postgres | postgres | pending | 127.0.0.1 | 55409 | 127.0.0.1  |       6432 |              |              | c65859260c452 |      |          0 | 
(2 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 pgbench  | postgres |         1 |          0 |         1 |       0 |       0 |         0 |        0 |       0 |          0 | session
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
(2 rows)

postgres=> show servers;
 type |   user   | database | state  |   addr    | port | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+--------+-----------+------+------------+------------+--------------+--------------+---------------+------+------------+-----
 S    | postgres | pgbench  | active | 127.0.0.1 | 1922 | 127.0.0.1  |      51473 |              |              | se2bb35408515 |      |          0 | 
(1 row)

可以发现在"session"模式下，与"transaction"模式不同，提交事务后数据库连接依然处于"active"状态。
同时这也说明"session"模式与"transaction"模式断开服务器连接的方式有所不同。

(5)退出这个会话

postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 55409 | 127.0.0.1  |       6432 |              |              | c65859260c452 |      |          0 | 
(1 row)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 pgbench  | postgres |         0 |          0 |         0 |       1 |       0 |         0 |        0 |       0 |          0 | session
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
(2 rows)

postgres=> show servers;
 type |   user   | database | state |   addr    | port | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+-------+-----------+------+------------+------------+--------------+--------------+---------------+------+------------+-----
 S    | postgres | pgbench  | idle  | 127.0.0.1 | 1922 | 127.0.0.1  |      51473 |              |              | se2bb35408515 |      |          0 | 
(1 row)

发现原来客户端已经没有了，但是原来的服务器连接还在，只不过是处于空闲的状态。

（6）等待pool_ttl空闲等待时间后

postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 55409 | 127.0.0.1  |       6432 |              |              | c65859260c452 |      |          0 | 
(1 row)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 pgbench  | postgres |         0 |          0 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | session
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
(2 rows)

postgres=> show servers;
 type | user | database | state | addr | port | local_addr | local_port | connect_time | request_time | ptr | link | remote_pid | tls 
------+------+----------+-------+------+------+------------+------------+--------------+--------------+-----+------+------------+-----
(0 rows)

发现此时服务器连接已经关闭。
```

### 预期测试结果:

​	数据库连接空闲超过60秒，数据库连接断开

### 测试结果：

​	数据库连接空闲超过60秒，数据库连接断开。

### 测试结论:

​	Odyssey的连接空闲超时时间测试成功

## 5、Odyssey启动测试

### 测试说明:

​	Odyssey连接池的启动不需要像pgbouncer那样，必须切换数据库用户下启动。

### 测试初始化:

​	脚本主要路由配置如下:

```
database "pgbench" {
	user "postgres" {
#		Authentication method.
#
#		"none"       - authentication turned off
#		"block"      - block this user
#		"clear_text" - PostgreSQL clear text authentication
#		"md5"        - PostgreSQL MD5 authentication
#		"cert"       - Compare client certificate Common Name against auth_common_name's
#
		authentication "none"

#
#		Authentication certificate CN.
#
#		Specify common names to check for "cert" authentification method.
#		If there are more then one common name is defined, all of them
#		will be checked until match.
#
#		Set 'default' to check for current user.
#
#		auth_common_name default
#		auth_common_name "test"

#
#		Authentication method password.
#
#		Depending on selected method, password can be in plain text or md5 hash.
#
#		password "111111"

#
#		Authentication query.
#
#		Use selected 'auth_query_db' and 'auth_query_user' to match a route.
#		Use matched route server to send 'auth_query' to get username and password needed
#		to authenticate a client.
#
#		auth_query "select username, pass from auth where username='%u'"
#		auth_query_db ""
#		auth_query_user ""

#
#		Client connections limit.
#
#		Comment 'client_max' to disable the limit. On client limit reach, Odyssey will
#		reply with 'too many connections'.
#
		client_max 200

#
#		Remote server to use.
#
#		By default route database and user names are used as connection
#		parameters to remote server. It is possible to override this values
#		by specifying 'storage_db' and 'storage_user'. Remote server password
#		can be set using 'storage_password' field.
#
		storage "postgres_server"
		storage_db "pgbench"
		storage_user "postgres"
		storage_password "123456"

#
#		Server pool mode.
#
#		"session"     - assign server connection to a client until it disconnects
#		"transaction" - assign server connection to a client during a transaction lifetime
#
		pool "session"

#
#		Server pool size.
#
#		Keep the number of servers in the pool as much as 'pool_size'.
#		Clients are put in a wait queue, when all servers are busy.
#
#		Set to zero to disable the limit.
#
#		pool_size 0

#
#		Server pool wait timeout.
#
#		Time to wait in milliseconds for an available server.
#		Disconnect client on timeout reach.
#
#		Set to zero to disable.
#
#		pool_timeout 0

#
#		Server pool idle timeout.
#
#		Close an server connection when it becomes idle for 'pool_ttl' seconds.
#
#		Set to zero to disable.
#
		pool_ttl 60

#
#		Server pool parameters discard.
#
#		Execute 'DISCARD ALL' and reset client parameters before using server
#		from the pool.
#
		pool_discard no

#
#		Server pool auto-cancel.
#
#		Start additional Cancel connection in case if server left with
#		executing query. Close connection otherwise.
#
		pool_cancel yes

#
#		Server pool auto-rollback.
#
#		Execute 'ROLLBACK' if server left in active transaction.
#		Close connection otherwise.
#
		pool_rollback yes

#
#		Forward PostgreSQL errors during remote server connection.
#
		client_fwd_error yes

#
#		Enable verbose mode for a specific route only.
#
		log_debug no
	}
}

```

### 测试过程:

```
(1)在root用户下启动:
[root@Odyssey sources]# /usr/local/src/odyssey/build/sources/odyssey /usr/local/src/odyssey/odyssey.conf 

查看Odyssey进程信息:
[postgres@Odyssey ~]$ ps -ef | grep odyssey
root      3736  1994  0 13:16 pts/0    00:00:00 /usr/local/src/odyssey/build/sources/odyssey /usr/local/src/odyssey/odyssey.conf
postgres  3742  2752  0 13:16 pts/1    00:00:00 grep odyssey

在root用户下启动成功

（2）在数据库用户下启动
[postgres@Odyssey ~]$  /usr/local/src/odyssey/build/sources/odyssey /usr/local/src/odyssey/odyssey.conf

查看Odyssey进程信息：
[postgres@Odyssey ~]$ ps -ef | grep odyssey
postgres  3780  3750  0 13:18 pts/0    00:00:00 /usr/local/src/odyssey/build/sources/odyssey /usr/local/src/odyssey/odyssey.conf
postgres  3784  2752  0 13:18 pts/1    00:00:00 grep odyssey
```

### 预期测试结果:

​	Odyssey在root用户和数据库用户下都可以启动Odyssey

### 测试结果:

​	Odyssey在root用户和数据库用户下都可以启动Odyssey

### 测试结论:

​	Odyssey启动测试通过

## 6、通过Odyssey连接数据库执行DML、DDL看是否生效

### 测试说明:

​	测试通过Odyssey连接数据库执行DML、DDL等操作，看数据更改是否生效。

### 测试初始化:

​	同Odyssey启动测试配置文件

### 测试过程:

```
[postgres@Odyssey ~]$ psql -p 6432 -h 127.0.0.1 -U postgres -d postgres
Password for user postgres: 
psql (10.3)
Type "help" for help.

postgres=# create table con_pool(id int,name text);
CREATE TABLE
postgres=# insert into con_pool(1,'a');                              ^
postgres=# insert into con_pool values (1,'a'); 
INSERT 0 1
postgres=# insert into con_pool values (2,'b'); 
INSERT 0 1
postgres=# update con_pool set name='c' where id=2;
UPDATE 1
postgres=# delete from con_pool where id=1;
DELETE 1
postgres=# select * from con_pool; 
 id | name 
----+------
  2 | c
(1 row)
```

### 预期测试结果:

​	通过Odyssey连接数据库所作的操作都落地

### 测试结果:

​	DDL、DML操作都生效

### 测试结论:

​	通过Odyssey连接数据库执行对数据库更改的操作都生效测试通过。

## 7、连接测试

### 测试说明:

​	测试通过Odyssey是否可以正常连接到指定的数据库

​	测试python是否可以通过Odyssey来连接到数据库

### 测试初始化:

​	参考Odyssey启动测试的配置文件

### 测试过程:

```
（1）测试通过Odyssey是否可以正常连接到指定的数据库
[postgres@Odyssey ~]$ psql -p 6432 -h 127.0.0.1 -U postgres -d postgres
Password for user postgres: 
psql (10.3)
Type "help" for help.

postgres=# 

 (2)测试python是否可以通过Odyssey来连接到数据库
 执行下面的脚本
#!/usr/bin/python
# -*- conding:utf-8 -*-
import  paramiko
import os
import sys
import psycopg2
import re
import time

def connectPostgreSQL(sql):
    conn = psycopg2.connect(database="postgres", user="postgres", password="111111", host="192.168.1.77", port="6432")
    print('connect successful!')
    cursor = conn.cursor()
    cursor.execute(sql)
    conn.commit()
    conn.close()


if __name__ == '__main__':
    connectPostgreSQL("drop table con_pool");
    
    
   登陆到数据库上发现python通过Odyssey连接到数据库把con_pool表删除了
   postgres=# \d
          List of relations
 Schema |   Name   | Type  |  Owner   
--------+----------+-------+----------
 public | con_pool | table | postgres
(1 row)

postgres=# \d
Did not find any relations.
```

### 预期测试结果:

​	通过Odyssey可以正常连接到指定的数据库

​	python可以通过Odyssey来连接到数据库

### 测试结果:

​	符合预期

### 测试结论:

​	数据库连接测试通过

8、

```

```

## 8、管理控制台测试

### 测试说明:

​	Odyssey中type参数有两个选项，"remote"表示PostgreSQL服务器而“local”表示管理控制台

测试Odyssey是否能通过设置type为”local“而达到和pgbouncer连接池中通过虚拟库pgbouncer来查看连接相关的信息。

### 测试初始化:

```
storage "postgres_server" {
        type "remote"
        host "127.0.0.1"
        port 1922
}

database "pgbench" {
        user "postgres" {
                authentication "none
                client_max 200
                storage "postgres_server"
                storage_db "pgbench"
                storage_user "postgres"
                storage_password "123456"
                pool "session"
                pool_ttl 60
                pool_discard no
                pool_cancel yes
                pool_rollback yes
                client_fwd_error yes
                log_debug no
        }
}

storage "postgres_server1" {
        type "local"
        host "127.0.0.1"
        port 1922
}


database "postgres" {
        user "postgres" {
                authentication "md5"
               password "111111"
                storage "postgres_server1"
                storage_db "postgres"
                storage_user "postgres"
                storage_password "123456"
                pool "transaction"
                pool_size 0
                pool_timeout 0
                pool_ttl 60
                pool_discard no
                pool_cancel yes
                pool_rollback yes
                client_fwd_error yes
                log_debug no
        }
}

```

### 测试过程:

```
连接到控制台
[postgres@Odyssey ~]$ psql -p 6432 -h 127.0.0.1 -U postgres -d postgres  
连接到PostgreSQL服务器
[root@python ~]# psql -p 6432 -h 192.168.1.77 -U postgres -d pgbench

在控制台查看Odyssey连接池的信息
postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         1 |          0 |         1 |       0 |       0 |         0 |        0 |       0 |          0 | session
(2 rows)

postgres=> show clients;
 type |   user   | database |  state  |     addr     | port  |  local_addr  | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+--------------+-------+--------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1    | 51530 | 127.0.0.1    |       6432 |              |              | c46642e42dfc4 |      |          0 | 
 C    | postgres | pgbench  | active  | 192.168.1.88 | 58055 | 192.168.1.77 |       6432 |              |              | c1916a62f8289 |      |          0 | 
(2 rows)

postgres=> 

```

### 预期测试结果:

​	Odyssey可以登陆到控制台执行show clients等命令查看

### 测试结果:

​	符合预期

### 测试结论:

​	Odyssey可以通过设置type为”local“而达到和pgbouncer连接池中通过虚拟库pgbouncer来查看连接相关的信息。

### 9、Odyssey中“session”模式和“transaction”模式的区别测试

这两种模式主要的差别在于服务器连接申请和释放的方式不同。(具体见测试项3、4)



### 10、Odyssey连接数据库参数生效场景测试

#### 测试说明:

​	Odyssey配置文件中有一个pool_discard参数，控制销毁连接池连接的参数，就是说在从连接池中使用连接之前，执行DISCARD ALL并重置客户端参数。(也就是如果复用了连接参数设置是否会变化)

#### 测试初始化：

​	该测试项我们将从“session”模式和“transaction”模式去测试在有无pool_discard参数时，配置PostgreSQL参数的时候是否会影响后面的连接。配置的参数就以work_mem为例。

#### 测试过程:

```
"session"模式下(pool_discard选项没有打开):
(1) 初始状态
postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | session
(2 rows)

postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 55445 | 127.0.0.1  |       6432 |              |              | c9ddfd628f16f |      |          0 | 
 C    | postgres | pgbench  | pending | 127.0.0.1 | 55446 | 127.0.0.1  |       6432 |              |              | ce758665d8746 |      |          0 | 
(2 rows)

postgres=> show servers;
 type | user | database | state | addr | port | local_addr | local_port | connect_time | request_time | ptr | link | remote_pid | tls 
------+------+----------+-------+------+------+------------+------------+--------------+--------------+-----+------+------------+-----
(0 rows)

（2）设置参数(如果做了操作就相当于连接到服务器连接了)、并查看状态
pgbench=# show work_mem;
 work_mem 
----------
 4MB
(1 row)

pgbench=# set work_mem to '8MB';
SET
pgbench=# show work_mem;
 work_mem 
----------
 8MB
(1 row)

postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 55445 | 127.0.0.1  |       6432 |              |              | c9ddfd628f16f |      |          0 | 
 C    | postgres | pgbench  | active  | 127.0.0.1 | 55446 | 127.0.0.1  |       6432 |              |              | ce758665d8746 |      |          0 | 
(2 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         1 |          0 |         1 |       0 |       0 |         0 |        0 |       0 |          0 | session
(2 rows)

postgres=> show servers;
 type |   user   | database | state  |   addr    | port | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+--------+-----------+------+------------+------------+--------------+--------------+---------------+------+------------+-----
 S    | postgres | pgbench  | active | 127.0.0.1 | 1922 | 127.0.0.1  |      51511 |              |              | sb9f5135d10e0 |      |          0 | 
(1 row)

（3）断开这个会话，在pool_ttl时间内复用这个数据库连接



postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 55445 | 127.0.0.1  |       6432 |              |              | c9ddfd628f16f |      |          0 | 
 C    | postgres | pgbench  | active  | 127.0.0.1 | 55452 | 127.0.0.1  |       6432 |              |              | cd26c2600ed25 |      |          0 | 
(2 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         1 |          0 |         1 |       1 |       0 |         0 |        0 |       0 |          0 | session
(2 rows)

postgres=> show servers;
 type |   user   | database | state  |   addr    | port | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+--------+-----------+------+------------+------------+--------------+--------------+---------------+------+------------+-----
 S    | postgres | pgbench  | active | 127.0.0.1 | 1922 | 127.0.0.1  |      51511 |              |              | sb9f5135d10e0 |      |          0 | 
 S    | postgres | pgbench  | idle   | 127.0.0.1 | 1922 | 127.0.0.1  |      51513 |              |              | s7274777b0077 |      |          0 | 
(2 rows)

发现此时复用了原来的连接

postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 55445 | 127.0.0.1  |       6432 |              |              | c9ddfd628f16f |      |          0 | 
 C    | postgres | pgbench  | active  | 127.0.0.1 | 55452 | 127.0.0.1  |       6432 |              |              | cd26c2600ed25 |      |          0 | 
(2 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         1 |          0 |         1 |       0 |       0 |         0 |        0 |       0 |          0 | session
(2 rows)

postgres=> show servers;
 type |   user   | database | state  |   addr    | port | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+--------+-----------+------+------------+------------+--------------+--------------+---------------+------+------------+-----
 S    | postgres | pgbench  | active | 127.0.0.1 | 1922 | 127.0.0.1  |      51511 |              |              | sb9f5135d10e0 |      |          0 | 
(1 row)

在复用客户端查看参数:
pgbench=# show work_mem;
 work_mem 
----------
 8MB
(1 row)

发现参数也没有重置。


（4）断开这个会话，在pool_ttl时间内没有这个数据库连接
等待pool_ttl时间后，确定这个连接真正的释放再去连接Odyssey

postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 55445 | 127.0.0.1  |       6432 |              |              | c9ddfd628f16f |      |          0 | 
(1 row)

postgres=> show servers;
 type | user | database | state | addr | port | local_addr | local_port | connect_time | request_time | ptr | link | remote_pid | tls 
------+------+----------+-------+------+------+------------+------------+--------------+--------------+-----+------+------------+-----
(0 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         0 |          0 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | session
(2 rows)

查看参数

[postgres@Odyssey ~]$ psql -p 6432 -h 127.0.0.1 -U postgres -d pgbench
psql (10.3)
Type "help" for help.

pgbench=# show work_mem;
 work_mem 
----------
 4MB
(1 row)

发现参数重置了。
查看Odyssey连接池状态发现local_port已经改变确实没有复用。

 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 55445 | 127.0.0.1  |       6432 |              |              | c9ddfd628f16f |      |          0 | 
 C    | postgres | pgbench  | active  | 127.0.0.1 | 55453 | 127.0.0.1  |       6432 |              |              | cfbc5a63895c3 |      |          0 | 
(2 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         1 |          0 |         1 |       0 |       0 |         0 |        0 |       0 |          0 | session
(2 rows)

postgres=> show servers;
 type |   user   | database | state  |   addr    | port | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+--------+-----------+------+------------+------------+--------------+--------------+---------------+------+------------+-----
 S    | postgres | pgbench  | active | 127.0.0.1 | 1922 | 127.0.0.1  |      51517 |              |              | s08f1e47a6e3f |      |          0 | 


```

```
"session"模式下(pool_discard选项打开)

（1）初始状态
postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 55458 | 127.0.0.1  |       6432 |              |              | cbe29116a3519 |      |          0 | 
 C    | postgres | pgbench  | pending | 127.0.0.1 | 55459 | 127.0.0.1  |       6432 |              |              | ca40fc43f3656 |      |          0 | 
(2 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | session
(2 rows)

postgres=> show servers;
 type | user | database | state | addr | port | local_addr | local_port | connect_time | request_time | ptr | link | remote_pid | tls 
------+------+----------+-------+------+------+------------+------------+--------------+--------------+-----+------+------------+-----
(0 rows)

（2)设置参数(如果做了操作就相当于连接到服务器连接了)、并查看状态

pgbench=# show work_mem;
 work_mem 
----------
 4MB
(1 row)

pgbench=# set work_mem to '8MB';
SET
pgbench=# show work_mem;
 work_mem 
----------
 8MB
(1 row)


postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 55458 | 127.0.0.1  |       6432 |              |              | cbe29116a3519 |      |          0 | 
 C    | postgres | pgbench  | active  | 127.0.0.1 | 55459 | 127.0.0.1  |       6432 |              |              | ca40fc43f3656 |      |          0 | 
(2 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         1 |          0 |         1 |       0 |       0 |         0 |        0 |       0 |          0 | session
(2 rows)

postgres=> show servers;
 type |   user   | database | state  |   addr    | port | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+--------+-----------+------+------------+------------+--------------+--------------+---------------+------+------------+-----
 S    | postgres | pgbench  | active | 127.0.0.1 | 1922 | 127.0.0.1  |      51524 |              |              | sce6a2f2b380b |      |          0 | 


（3）断开这个会话，在pool_ttl时间内复用这个数据库连接
postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 55458 | 127.0.0.1  |       6432 |              |              | cbe29116a3519 |      |          0 | 
 C    | postgres | pgbench  | pending | 127.0.0.1 | 55462 | 127.0.0.1  |       6432 |              |              | c7c4c0e675fb9 |      |          0 | 
(2 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         0 |          1 |         0 |       1 |       0 |         0 |        0 |       0 |          0 | session
(2 rows)

postgres=> show servers;
 type |   user   | database | state |   addr    | port | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+-------+-----------+------+------------+------------+--------------+--------------+---------------+------+------------+-----
 S    | postgres | pgbench  | idle  | 127.0.0.1 | 1922 | 127.0.0.1  |      51524 |              |              | sce6a2f2b380b |      |          0 | 
(1 row)

postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 55458 | 127.0.0.1  |       6432 |              |              | cbe29116a3519 |      |          0 | 
 C    | postgres | pgbench  | active  | 127.0.0.1 | 55462 | 127.0.0.1  |       6432 |              |              | c7c4c0e675fb9 |      |          0 | 
(2 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         1 |          0 |         1 |       0 |       0 |         0 |        0 |       0 |          0 | session
(2 rows)

postgres=> show servers;
 type |   user   | database | state  |   addr    | port | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+--------+-----------+------+------------+------------+--------------+--------------+---------------+------+------------+-----
 S    | postgres | pgbench  | active | 127.0.0.1 | 1922 | 127.0.0.1  |      51524 |              |              | sce6a2f2b380b |      |          0 | 


pgbench=# show work_mem;
 work_mem 
----------
 4MB
(1 row)

发现连接虽然复用了但是参数却重置了。
```

```
"transaction"模式下(没有打开pool_discard)：
（1）初始状态(已经再一个会话中设置了参数)
postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 55464 | 127.0.0.1  |       6432 |              |              | c592c5e22fd03 |      |          0 | 
 C    | postgres | pgbench  | active  | 127.0.0.1 | 55465 | 127.0.0.1  |       6432 |              |              | c032a64757377 |      |          0 | 
(2 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         1 |          0 |         1 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
(2 rows)

postgres=> show servers;
 type |   user   | database | state  |   addr    | port | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+--------+-----------+------+------------+------------+--------------+--------------+---------------+------+------------+-----
 S    | postgres | pgbench  | active | 127.0.0.1 | 1922 | 127.0.0.1  |      51530 |              |              | s953c581abcb3 |      |          0 | 
(1 row)

pgbench=# start transaction;
START TRANSACTION
pgbench=# show work_mem;
 work_mem 
----------
 4MB
(1 row)

pgbench=# set work_mem to '8MB';
SET
pgbench=# show work_mem;
 work_mem 
----------
 8MB
(1 row)

注意还没有提交

(2)开启另外一个会话，等待之前会话提交事务，然后在pool_ttl之前复用连接

pgbench=# start transaction;
START TRANSACTION
pgbench=# show work_mem;
 work_mem 
----------
 8MB
(1 row)

发现参数没有重置

（3）开启另外一个会话，等待之前会话提交事务，然后在pool_ttl之前不复用连接
pgbench=# start transaction;
START TRANSACTION
pgbench=# show work_mem;
 work_mem 
----------
 4MB
(1 row)
```

```
"transaction"模式下(打开pool_discard)：
(1)初始状态(已经再一个会话中设置了参数)


postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 55471 | 127.0.0.1  |       6432 |              |              | ceb604b172792 |      |          0 | 
 C    | postgres | pgbench  | active  | 127.0.0.1 | 55472 | 127.0.0.1  |       6432 |              |              | ce56d7c0dad6a |      |          0 | 
 C    | postgres | pgbench  | pending | 127.0.0.1 | 55475 | 127.0.0.1  |       6432 |              |              | cb88cc200226e |      |          0 | 
(3 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         1 |          1 |         1 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
(2 rows)

postgres=> show servers;
 type |   user   | database | state  |   addr    | port | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+--------+-----------+------+------------+------------+--------------+--------------+---------------+------+------------+-----
 S    | postgres | pgbench  | active | 127.0.0.1 | 1922 | 127.0.0.1  |      51537 |              |              | s979cfd6d9660 |      |          0 | 
(1 row)

pgbench=# start transaction;
START TRANSACTION
pgbench=# show work_mem;
 work_mem 
----------
 4MB
(1 row)

pgbench=# set work_mem to '8MB';
SET
pgbench=# show work_mem;
 work_mem 
----------
 8MB
(1 row)

注意还没有提交

(2)开启另外一个会话，等待之前会话提交事务，然后在pool_ttl之前复用连接

pgbench=# start transaction;
START TRANSACTION
pgbench=# show work_mem;
 work_mem 
----------
 4MB
(1 row)
```

#### 预期测试结果:

​	如果复用了连接，pool_discard设置为yes,则之前修改的参数不会变。

​	如果设置为no的话，之前修改的参数全部重置。

#### 测试结果:

​	符合预期结果

#### 测试结论：

​	pool_discard测试通过

### 11、连接自动回滚操作

#### 测试说明:

​	设置连接是否自动回滚，该参数为pool_rollback。若设置为“yes”，那么如果在活动的事务中连接断开，就自动执行回滚操作，否则关闭连接。

#### 测试初始化:

​	将pool_rollback参数设置为yes

​	将pool设置为transaction模式

#### 测试过程:

```
1、初始状态

postgres=> show clients;
 type |   user   | database |  state  |   addr    | port  | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+---------+-----------+-------+------------+------------+--------------+--------------+---------------+------+------------+-----
 C    | postgres | postgres | pending | 127.0.0.1 | 60334 | 127.0.0.1  |       6432 |              |              | c7b768f5422bd |      |          0 | 
 C    | postgres | pgbench  | pending | 127.0.0.1 | 60335 | 127.0.0.1  |       6432 |              |              | ce8799f141362 |      |          0 | 
(2 rows)

postgres=> show pools;
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode  
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 postgres | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbench  | postgres |         0 |          1 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | transaction
(2 rows)

postgres=> show servers;
 type |   user   | database | state  |   addr    | port | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+--------+-----------+------+------------+------------+--------------+--------------+---------------+------+------------+-----
 S    | postgres | pgbench  | active | 127.0.0.1 | 1922 | 127.0.0.1  |      55281 |              |              | sbd69ca0bdb28 |      |          0 | 
(1 row)

此时已经在另外一个会话中进行了如下的操作

pgbench=# start transaction;
START TRANSACTION
pgbench=# create table test(id int,name text);
CREATE TABLE
pgbench=# insert into test values(1,'a');
INSERT 0 1
pgbench=# insert into test values(2,'b');
INSERT 0 1

(2)断开服务器连接

ps -ef
postgres  5053  1981  0 13:01 ?        00:00:00 postgres: checkpointer process   
postgres  5054  1981  0 13:01 ?        00:00:00 postgres: writer process      
postgres  5055  1981  0 13:01 ?        00:00:00 postgres: wal writer process   
postgres  5056  1981  0 13:01 ?        00:00:00 postgres: autovacuum launcher process   
postgres  5057  1981  0 13:01 ?        00:00:00 postgres: stats collector process   
postgres  5058  1981  0 13:01 ?        00:00:00 postgres: bgworker: logical replication launcher   
postgres  5062  1981  0 13:02 ?        00:00:00 postgres: postgres pgbench 127.0.0.1(55281) idle

kill -9 5062

postgres=> show servers;
 type | user | database | state | addr | port | local_addr | local_port | connect_time | request_time | ptr | link | remote_pid | tls 
------+------+----------+-------+------+------+------------+------------+--------------+--------------+-----+------+------------+-----
(0 rows)

发现此时数据库连接已经关闭

(3)查看连接是否自动回滚

pgbench=# select * from test;
ERROR:  relation "test" does not exist
LINE 1: select * from test;

发现事务确实已经回滚。
```

#### 预期测试结果:

​	如果在活动的事务中连接断开，就自动执行回滚操作，否则关闭连接。

#### 测试结果:

​	符合预期

#### 测试结论:

​	连接自动回滚测试通过

### 12、连接自动取消测试

#### 测试说明:

​	设置连接是否自动取消,该参数为pool_cancel。若设置为“yes”，那么万一在执行查询的时候连接断开，就自动取消连接，否则关闭连接.

#### 测试初始化：

​	pool_cancel设置为yes

​	pool设置为“transaction”模式

#### 测试过程:

```
（1）初始状态
pgbench=# select * from pgbench_accounts;

postgres=> show servers;
 type |   user   | database | state  |   addr    | port | local_addr | local_port | connect_time | request_time |      ptr      | link | remote_pid | tls 
------+----------+----------+--------+-----------+------+------------+------------+--------------+--------------+---------------+------+------------+-----
 S    | postgres | pgbench  | active | 127.0.0.1 | 1922 | 127.0.0.1  |      55283 |              |              | s9ee8da513f3b |      |          0 | 
(1 row)

(2)模拟在查询中断开服务器连接
ps -ef

postgres  5053  1981  0 13:01 ?        00:00:00 postgres: checkpointer process   
postgres  5054  1981  0 13:01 ?        00:00:00 postgres: writer process      
postgres  5055  1981  0 13:01 ?        00:00:00 postgres: wal writer process   
postgres  5056  1981  0 13:01 ?        00:00:00 postgres: autovacuum launcher process   
postgres  5057  1981  0 13:01 ?        00:00:00 postgres: stats collector process   
postgres  5058  1981  0 13:01 ?        00:00:00 postgres: bgworker: logical replication launcher   
postgres  5131  1981  0 13:19 ?        00:00:00 postgres: postgres pgbench 127.0.0.1(55283) SELECT
postgres  5132  4884  0 13:19 pts/1    00:00:00 psql -p 6432 -h 127.0.0.1 -U postgres -d postgres
postgres  5133  4940  0 13:19 pts/3    00:00:00 psql -p 6432 -h 127.0.0.1 -U postgres -d pgbench
root      5142  4987  0 13:20 pts/4    00:00:00 ps -ef


[root@Odyssey ~]# kill -9 5131

(3)查看服务器状态

postgres=> show servers;
 type | user | database | state | addr | port | local_addr | local_port | connect_time | request_time | ptr | link | remote_pid | tls 
------+------+----------+-------+------+------+------------+------------+--------------+--------------+-----+------+------------+-----
(0 rows)

发现服务器连接确实已经取消，如果是在正常的状态下释放服务器连接，show servers会发现state为idle状态，不会马上回收，而是等待一个pool_ttl超时时间。
```

#### 预期测试结果:

​	在执行查询的时候连接断开，就自动取消连接，否则关闭连接.

#### 测试结果:

​	符合预期

#### 测试结论:

​	连接自动取消测试通过。

### 13、身份验证

```
"none"       - authentication turned off
"block"      - block this user
"clear_text" - PostgreSQL clear text authentication
"md5"        - PostgreSQL MD5 authentication
"cert"       - Compare client certificate Common Name against auth_common_name's
```

1. none（不需要认证）

部分参数设置：

```
database "postgres" {
         user default {
                authentication "none"
                password "postgres"
                client_max 10000
                storage "tpcb"
                storage_db "postgres"
                storage_user "postgres"
                storage_password "postgres"
```

登录不需要认证：

```
[postgres@sdw-15 ~]$ psql -p 7432 -d postgres
psql (11.1)
Type "help" for help.
```

2. block（锁定，不能登录）

   部分参数设置：

   ```
   database "postgres" {
            user default {
                   authentication "block"
                   password "postgres"
                   client_max 10000
                   storage "tpcb"
                   storage_db "postgres"
                   storage_user "postgres"
                   storage_password "postgres"
   ```

   登录报错：

   ```
   [postgres@sdw-15 ~]$ psql -p 7432 -d postgres
   psql: ERROR:  odyssey: c4ba15d1010ec: user blocked
   ```

   日志：

   ```
   28 Feb 12:07:00.017, none,none, ::1:12634, 40601, info [c4ba15d1010ec none], (startup), new client connection [::1]:12634 
   28 Feb 12:07:00.018, postgres,postgres, ::1:12634, 40601, info [c4ba15d1010ec none], (startup), route 'postgres.postgres' to 'postgres.default' 
   28 Feb 12:07:00.018, postgres,postgres, ::1:12634, 40601, info [c4ba15d1010ec none], (auth), user 'postgres.postgres' is blocked 
   ```

   3. clear_text（明文身份验证认证）

   参数设置：

   ```
   database "postgres" {
            user default {
                   authentication "clear_text"
                   password "postgres"
                   client_max 10000
                   storage "tpcb"
                   storage_db "postgres"
                   storage_user "postgres"
                   storage_password "postgres"
   ```

   登录需要输入密码：

   ```
   postgres@sdw-15 ~]$ psql -p 7432 -d postgres
   Password for user postgres: 
   psql (11.1)
   Type "help" for help.
   ```

   4. md5（md5认证）

   参数设置：

   ```
   database "postgres" {
            user default {
                   authentication "md5"
                   password "postgres"
                   client_max 10000
                   storage "tpcb"
                   storage_db "postgres"
                   storage_user "postgres"
                   storage_password "postgres"
   ```

   登录需要输入密码：

   ```
   postgres@sdw-15 ~]$ psql -p 7432 -d postgres
   Password for user postgres: 
   psql (11.1)
   Type "help" for help.
   ```

   5. cert（将客户端证书的公共名称与auth_common_name进行比较）

   参数设置：

   ```
   database "postgres" {
            user default {
                   authentication "cert"
                   password "postgres"
                   client_max 10000
                   storage "tpcb"
                   storage_db "postgres"
                   storage_user "postgres"
                   storage_password "postgres"
   ```

   登录失败：

   ```
   [postgres@sdw-15 ~]$ psql -p 7432 -d postgres -h127.0.01 -Upostgres
   psql: ERROR:  odyssey: c6b8f8b32799c: TLS connection required
   ```

   