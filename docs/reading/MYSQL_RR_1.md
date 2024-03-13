# RR真的解决幻读了吗?
在RR级别下执行以下两个事务
```sql
start TRANSACTION;
SELECT * FROM TABLE LIMIT 10;
SELECT * FROM TABLE LIMIT 10;
COMMIT ;
```
```sql
start TRANSACTION;
INSERT INTO TABLE (id) VALUES (1);
COMMIT ;
```
在第二次执行查询之前，执行事务2，发现确实读到的还是之前的数据，并没有变化,我们修改下事务1的sql,给第二次查询加上共享锁，或者排他锁也可以
```sql
start TRANSACTION;
SELECT * FROM TABLE LIMIT 10;
SELECT * FROM TABLE LIMIT 10 LOCK IN SHARE MODE;
-- SELECT * FROM TABLE LIMIT 10 FOR UPDATE;
COMMIT ;
```
重复刚才的步骤，会发现第二次读取，读到了插入的数据
# 总结
1. mysql使用mvcc版本号的方式实现快照读，来保证可重复读的特性
2. mysql的RR隔离级别使用next-key-lock(间隙锁+行锁)来避免幻读,比如将两个查询调换个位置，插入的事务是无法执行的，会一直阻塞到锁释放
```sql
start TRANSACTION;
SELECT * FROM TABLE LIMIT 10 LOCK IN SHARE MODE;
SELECT * FROM TABLE LIMIT 10;
COMMIT ;
```
基于以上两点，似乎认为mysql的innodb已经解决了幻读，<font color = #FF000 size=3 >但是间隙锁只会在加了S锁/X锁的情况下才会加上间隙锁</font>而mysql的读默认都是读快照，在快照读的情况下，读取的是历史数据,看似解决了幻读的问题，实际上因为没有加S锁/X锁，会导致没有加间隙锁，<font color = #FF000 size=3 >假如这个时候，插入了一条数据，再次查询，这时候查询加锁，不会读快照，而是读当前，就会出现幻读的情况</font>