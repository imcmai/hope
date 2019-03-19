## 背景
**公司部署了BI报表系统，每天脚本同步生产数据到报表的库，报表在单独的服务器上，mysql的CPU占用率最高可达百分之600左右， 由于报表相对来说，一天之内数据不会发生改变，不会发生除了查询之外的增删改操作，所以缓存的命中是很简单的, 所以开启查询缓存， 目前cpu占用率缩减十多倍** 
- ### 缓存的解释:
    - 在进行一次查询时,如果执行的查询和上次一致则直接返回上次的结果集,如果在此期间，进行了写操作，则缓存失效不会命中
    - 如果查询使用了now之类的实时函数也不会进行缓存
- ### 启用查询缓存
```sql
      SHOW VARIABLES LIKE '%query_cache%'; 查询当前缓存开启状态
      have_query_cache:是否使用缓存
      query_cache_limit:单个查询使用的缓冲区大小
      query_cache_min_res_unit:最小缓存块大小(数据量大可以相应调整变大)
      query_cache_size:缓存区大小
      query_cache_type:缓存开启状态0或off关闭缓存 
         1或on开启缓存，但是不保存使用sql_no_cache的select语句,如不缓存select  sql_no_cache name from wei where id=2 
         2或demand开启有条件缓存，只缓存带sql_cache的select语句，缓存select  sql_cache name from wei where id=4 
      query_cache_wlock_invalidate:其他客户端进行写操作时，如果有客户端命中缓存 ，是否直接返回cache结果，还是等待写入完毕
    - Cachetype默认是未开启状态，去服务器找到MYSQL的配置文件(my.conf)，编辑mysqlld下的配置，把cachetype设置为1，重启mysql
```
```sql
    show status like 'qcache%' 显示当前缓存状况
    -- Qcache_free_blocks：缓存中相邻内存块的个数。数目大说明可能有碎片。
    -- Qcache_free_memory：缓存中的空闲内存。
    -- Qcache_hits：每次查询在缓存中命中时就增大
    -- Qcache_inserts：每次插入一个查询时就增大。命中次数除以插入次数就是不中比率。
    -- Qcache_lowmem_prunes：缓存出现内存不足并且必须要进行清理以便为更多查询提供空间的次数。-- --
    -- 这个数字最好长时间来看;如果这个 数字在不断增长，就表示可能碎片非常严重，或者内存很少。
    -- (上面的 free_blocks和free_memory可以告诉您属于哪种情况)
    -- Qcache_not_cached：不适合进行缓存的查询的数量，通常是由于这些查询不是 SELECT 语句或者用了now()之类的函数。
    -- Qcache_queries_in_cache：当前缓存的查询(和响应)的数量。
    -- Qcache_total_blocks：缓存中块的数量。
     FLUSH QUERY CACHE ; 清理缓存碎片
 ```
    
 
 
 

 