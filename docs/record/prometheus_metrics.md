## mysql
|告警策略 |参考指标 |
| :--- | :----------- |
| mysql慢查询最近一分钟大于0 | mysql_global_status_slow_queries |
| 30分钟内表锁比例大于30% | mysql_global_status_table_locks_immediate行锁，mysql_global_status_table_locks_waited表锁 |
| mysql连接数超过80% | mysql_global_status_threads_connected连接数，mysql_global_variables_max_connections最大连接数 |
| 实例宕机 | mysql_up |
## jvm
|告警策略 |参考指标 |
| :--- | :----------- |
| 最近三十分钟error日志有新增 | log4j2_events_total{level="error"} |
| 元空间超过500M、800M、1024M | jvm_memory_used_bytes{area="nonheap",id="Metaspace"} |
| 一小时内major gc超过三次 | jvm_gc_pause_seconds_count{action="end of major GC"} |
| 一分钟内minor gc超过30次 | jvm_gc_pause_seconds_count{action="end of minor GC"} |
| 堆内存超过80% | jvm_memory_used_bytes{area="heap"} |

