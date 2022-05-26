# MYSQL事务

事务是数据库区别文件系统重要特性之一。

ACID全称：原子性（atomicity）、一致性（consistency）、隔离性（isolation）、持久性（durability）。

## 1. 分类

*   扁平事务

    所有操作处于同一层次，由beginwork开始，rollback或commit结束，其间的操作是原子性的。
* 带有保存点的事务
*   嵌套事务

    嵌套事务是一个层次结构框架。由一个顶层事务控制各个层次事务。
* 分布式事务

## 2.实现

事务的隔离性由锁机制来实现。原子性、一致性、持久性通过数据库的redo log和undo log来完成。redo log称为重做日志，用来保证事务原子性、持久性。undo log用来保证事务一致性。redo和undo的作用都可以视为一种恢复操作，redo恢复提交事务修改的页操作，undo回滚到行记录某个特定版本。 因此两者内容不同，redo通常是物理日志，记录的是页的物理操作，undo是逻辑日志，根据每行记录进行记录，帮助事务回滚和MVCC功能。

### 2.1 redo

重做日志用来实现事务的持久性。其由两部分组成：一是内存中的重做日志缓冲，其是易失的；二是重做日志文件，其是持久的。 事务提交时，必须将该事务的所有日志（redo| undo）写入到重做日志文件进行持久化，待事务的commit操作才算完成。

### 2.2 undo

重做日志记录了事务的行为，但是事务有时还需要进行回滚操作，这时就需要undo。undo段存在共享表空间内

## 3. 隔离级别

READ UNCOMMITTED READ COMMITTED REPEATABLE READ SERIALIZABLE

## 4. 分布式事务

HA事务：隔离级别必须设置为SERIALIZABLE

## 5. 不好的事务习惯

循环提交事务 事务回滚（需要抛除异常）