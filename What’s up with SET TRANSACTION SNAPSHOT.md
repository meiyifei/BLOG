## What’s up with SET TRANSACTION SNAPSHOT?

11 February 2019

A feature of PostgreSQL that most people don’t even know exists is the ability to export and import transaction snapshots.

PostgreSQL的一个大多数人甚至不知道存在的功能是导出和导入事务快照的能力。 

The documentation is accurate, but it doesn’t really describe *why* one might want to do such a thing.

文档是准确的，但它并没有真正描述*为什么*人们可能想做这样的事情 

First, what is a “snapshot”? You can think of a snapshot as the current set of committed tuples in the database, a consistent view of the database. When you start a transaction and set it to `REPEATABLE READ`mode, the snapshot remains consistent throughout the transaction, even if other sessions commit transactions. (In the default transaction mode, `READ COMMITTED`, each statement starts a new snapshot, so newly committed work could appear between statements within the transaction.)

首先，什么是“快照”？ 您可以将快照视为数据库中当前提交的元组集，即数据库的一致视图。 当您启动事务并将其设置为`REPEATABLE READ`模式时，即使其他会话提交事务，快照在整个事务中也保持一致。 （在默认事务模式下， `READ COMMITTED` ，每个语句都会启动一个新快照，因此新提交的工作可能出现在事务中的语句之间。） 

However, each snapshot is local to a single transaction. But suppose you wanted to write a tool that connected to the database in multiple sessions, and did analysis or extraction? Since each session has its own transaction, and the transactions start asynchronously from each other, they could have different views of the database depending on what other transactions got committed. This might generate inconsistent or invalid results.

但是，每个快照都是单个事务的本地快照。 但是假设您想编写一个在多个会话中连接到数据库的工具，并进行分析或提取？ 由于每个会话都有自己的事务，并且事务彼此异步启动，因此它们可能具有不同的数据库视图，具体取决于提交的其他事务。 这可能会产生不一致或无效的结果。 

This isn’t theoretical: Suppose you are writing a tool like `pg_dump`, with a parallel dump facility. If different sessions got different views of the database, the resulting dump would be inconsistent, which would make it useless as a backup tool!

这不是理论上的：假设您正在使用并行转储工具编写类似`pg_dump`工具。 如果不同的会话获得了数据库的不同视图，则生成的转储将不一致，这将使其无法用作备份工具！ 

The good news is that we have the ability to “synchronize” various sessions so that they all use the same base snapshot.

好消息是我们能够“同步”各种会话，以便它们都使用相同的基本快照。 

First, a transaction opens and sets itself to `REPEATABLE READ` or `SERIALIZABLE` mode (there’s no point in doing exported snapshots in `READ COMMITTED` mode, since the snapshot will get replaced at the very next statement). Then, that session calls [`pg_export_snapshot`](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-SNAPSHOT-SYNCHRONIZATION). This creates an identifier for the current transaction snapshot.

首先，事务打开并将自身设置为`REPEATABLE READ`或`SERIALIZABLE`模式（在`READ COMMITTED`模式下执行导出快照没有意义，因为快照将在下一个语句中被替换）。 然后，该会话调用[`pg_export_snapshot`](https://translate.googleusercontent.com/translate_c?depth=1&hl=zh-CN&prev=search&rurl=translate.google.com.hk&sl=en&sp=nmt4&u=https://www.postgresql.org/docs/current/functions-admin.html&xid=17259,15700022,15700186,15700191,15700248,15700253&usg=ALkJrhgUv6ivQN5CHKyGJGWxJSTr3vfmNQ#FUNCTIONS-SNAPSHOT-SYNCHRONIZATION) 。 这将为当前事务快照创建标识符。 

Then, the client running the first session passes that identifier to the clients that will be using it. You’ll need to do this via some non-database channel. For example, you can’t use `LISTEN` / `NOTIFY`, since the message isn’t actually sent until `COMMIT` time.

然后，运行第一个会话的客户端将该标识符传递给将使用它的客户端。 您需要通过一些非数据库通道执行此操作。 例如，您不能使用`LISTEN` / `NOTIFY` ，因为直到`COMMIT`时间才会实际发送消息。 

Each client that receives the snapshot ID can then do [`SET TRANSACTION SNAPSHOT ...`](https://www.postgresql.org/docs/current/sql-set-transaction.html) to use the snapshot. The client needs to call this before it does any work in the session (even `SELECT`). Now, each of the clients has the same view into the database, and that view will remain until it `COMMIT`s or `ABORT`s.

然后，每个接收快照ID的客户端都可以执行[`SET TRANSACTION SNAPSHOT ...`](https://translate.googleusercontent.com/translate_c?depth=1&hl=zh-CN&prev=search&rurl=translate.google.com.hk&sl=en&sp=nmt4&u=https://www.postgresql.org/docs/current/sql-set-transaction.html&xid=17259,15700022,15700186,15700191,15700248,15700253&usg=ALkJrhhUOph6TrpM9FjQedN1gt9k6oMeGg)以使用快照。 客户端需要在会话中执行任何工作之前调用它（即使是`SELECT` ）。 现在，每个客户端对数据库具有相同的视图，并且该视图将保持到`COMMIT`或`ABORT`为止。 

Note that each transaction is still fully autonomous; the various sessions are not “inside” the same transaction. They can’t see each other’s work, and if two different clients modify the database, those modifications are not visible to any other session, including the ones that are sharing the snapshot. You can think of the snapshot as the “base” view of the database, but each session can modify it (subject, of course, to the usual rules involved in modifying the same tuples, or getting serialization failures).

请注意，每个事务仍然是完全自治的; 各种会话不在同一交易的“内部”。 他们无法看到彼此的工作，如果两个不同的客户端修改数据库，那么任何其他会话（包括共享快照的会话）都不会看到这些修改。您可以将快照视为数据库的“基本”视图，但每个会话都可以对其进行修改（当然，主题是修改相同元组或获取序列化失败所涉及的常规规则）。 

This is a pretty specialized use-case, of course; not many applications need to have multiple sessions with a consistent view of the database. But if you do, PostgreSQL has the facilities to do it!

 当然，这是一个非常专业的用例; 没有多少应用程序需要具有一致的数据库视图的多个会话。 但是，如果你这样做，PostgreSQL可以做到这一点！ 