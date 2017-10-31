# 实施oracle数据库的审计

## Locks（锁）

- Prevent multiple sessions from changing the same data at the same time.
- Are automatically obtained at the lowest possible level for a given statuement.
- Do not escalate(逐步升级).  
数据库的基本功能，防止数据的不一致。

## 表的级别

- 表级锁：针对整个表进行加锁。
- 行级锁：只针对表中行进行加锁。
- 排他锁：一个事物对一个表(行)进行加锁之后，那么其他事务就不能加同样的锁。
    + DDL: ALTER DROP (表)
    + DML: UPDATE DELETE (行)
- 共享锁：当一个事物执行时对表(行)进行操作时加锁，其它事务也可对此表(行)进行加锁。
    + DML: SELECT

## Locking Mechanism

- High level of data concurrency:  
    + Row-level locks for inserts,updates,and deletes
    + No locks required for queries
- Automatic queue management
- Locks held until the transaction ends (with the **COMMIT** or **ROLLBACK** operation.)

## Possible Causes of Lock Conflicts

- Uncommitted changes
- Long-running transactions
- Unnecessarily high locking levels

## Detecting Lock Conflicts & Resolving Lock Conflicts With SQL

- 使用如下sql语句检查数据库是否有锁冲突
```sql
SQL> run
  1  select username,sid,serial#,blocking_session from v$session
  2* where username is not null

USERNAME                              SID    SERIAL# BLOCKING_SESSION
------------------------------ ---------- ---------- ----------------
SCOTT                                 146      41283
SYS                                   162       2307
SCOTT                                 194       5263              146
```

- 使用如下sql语句终止一个blocking的session
```sql
SQL> alter system kill session '146,41283';  
```

## Deadlocks(死锁) & Resolving Deadlocks

- 两个事务互相等待对方释放资源，oracle认定为产生了死锁，在这种情况下，将以牺牲一个事务作为代价，另一个事务继续执行，牺牲的事务将回滚。
- ORA-00060的错误并记录在数据库的日志文件alertSID.log中。同时在user_dump_dest下产生一个跟踪文件，详细描述死锁的相关信息。

### Resolving 

- 执行commit或者rollback结束事务
- 终止会话`(alter system kill session 'SID,SERIAL#')`


## Managing Undo data

Undo data is:  

- A copy of original, premodified data
- Captured for every transaction that changes data
- Retained at least until the transaction id ended
- Used to support:
    + Rollback operations
    + Read-consistent queries
    + Oracle Flashback Query, Oracle Flashback Transaction, and Oracle Flashback Table
    + Recovery from failed transactions

为了保证在一次事务中读取数据的一致性，undo数据在undo表空间中默认保留(Retained)900秒。

查询表空间：  
```sql
SQL> select tablespace_name,contents
  2  from dba_tablespaces;

TABLESPACE_NAME                CONTENTS
------------------------------ ---------
SYSTEM                         PERMANENT
SYSAUX                         PERMANENT
UNDOTBS1                       UNDO
TEMP                           TEMPORARY
USERS                          PERMANENT
XLSGRID                        PERMANENT

6 rows selected.
```

查询数据库当前使用的undo表空间：
```sql
SQL> show parameter undo_tablespace

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
undo_tablespace                      string      UNDOTBS1
```

查询undo数据段再事务提交后的保存时间(单位：秒)：
```sql
SQL> show parameter undo_retention;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
undo_retention                       integer     900
```

    Ps：一般情况下不会让UNDO表空间自动扩展，因为不可控。

### Managing Undo

Automatic undo management:  

- Fully automated management of undo data and space in a dedicated undo tablespace
- For all sessions
- Self-tuning in AUTOEXTEND tablespaces to satisfy long-runging queries
- Self-tunning in fixed-size tablespaces for best retention

DBA tasks in support of Flashback operations:  

- Configuring undo retention
- Changing undo tablespace to a fixed size
- Avoiding space and "snapshot too old" errors

#### Configuring undo retention：

- Guaranteeing Undo Retention(undo数据的保留时间要得到保证，默认当undo表空间用完时，自动覆盖)

```sql
SQL> run   //查询保留是否得到保证
  1  select tablespace_name,contents,retention from dba_tablespaces
  2* where tablespace_name = 'UNDOTBS1'

TABLESPACE_NAME                CONTENTS  RETENTION
------------------------------ --------- -----------
UNDOTBS1                       UNDO      NOGUARANTEE

SQL> ALTER TABLESPACE undotbs1 RETENTION GUARANTEE; //开启保证
```


