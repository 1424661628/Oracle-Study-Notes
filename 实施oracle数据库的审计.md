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