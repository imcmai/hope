## 背景
我们提供了容器化的mysql给业务组使用， 容器化的意义在于技术栈统一，提供了标准的账号权限体系、配置文件优化、定时备份、主从切换、指标监控、异常告警。

这次的问题就出在配置文件优化上，以下是我们提供在2核4G环境下的一个模板，参考了一些mysql调优的案例和文章
```code
  [mysqld]
    character_set_server            = utf8mb4
    datadir                         = /var/lib/mysql
    expire_logs_days                = 1
    explicit_defaults_for_timestamp = 1
    general_log                     = ON
    general_log_file                = /var/log/mysql/general_log_file.log
    innodb_buffer_pool_size         = 2G
    innodb_flush_log_at_trx_commit  = 2
    innodb_flush_method             = O_DIRECT
    innodb_flush_neighbors          = 0
    innodb_io_capacity              = 1000
    innodb_io_capacity_max          = 2000
    innodb_large_prefix             = 1
    innodb_lock_wait_timeout        = 30
    innodb_print_all_deadlocks      = 1
    innodb_thread_concurrency       = 4
    join_buffer_size                = 20M
    log-error                       = /var/log/mysql/error.log
    log_queries_not_using_indexes   = 0
    log_slow_admin_statements       = 1
    log_slow_slave_statements       = 1
    log_timestamps                  = system
    long_query_time                 = 10
    max_connect_errors              = 10
    pid-file                        = /var/run/mysqld/mysqld.pid
    read_rnd_buffer_size            = 8388608
    slow_query_log                  = 1
    slow_query_log_file             = /var/log/mysql/slow_query_log_file.log
    socket                          = /var/run/mysqld/mysqld.sock
    sort_buffer_size                = 4194304
    tmp_table_size                  = 67108864
    wait_timeout                    = 600
    binlog-ignore-db                = mysql
    enforce-gtid-consistency        = ON
    gtid-mode                       = ON
    log-bin                         = mysql-bin
    log-slave-updates               = ON
    innodb_log_file_size            = 256M
    lower_case_table_names          = 1
    max_connections                 = 512
    server-id                       = 1
```
## 问题描述
原来的数据库在虚拟机上运行，8核16G，需要迁移到容器上来， 在迁移测试的过程中， 突然发现所有的表都打不开了，执行 ```show full processlist```发现有四个视图语句在执行，询问得知数仓会在这个时间抽数，并发执行这几个视图语句，但是之前在虚拟机上跑是没问题的
## 排查
### 磁盘
刚开始以为是磁盘的问题，容器化的mysql，磁盘是挂载的pv、pvc，用的ceph的这一套分布式存储，通过ceph的性能测试和控制面板看起来是没有任何问题的，我们每天上亿的日志采集走的也是ceph，但是为了宁杀错不放过，将容器化的mysql磁盘挂载到了虚拟机上的固态硬盘， 还是没有解决这个问题
### 网络
磁盘没有问题，从网络入手，视图的查询，走的是k8s的nodeport，怀疑可能是网络传输过程中丢包导致的，于是将视图的查询搬到了容器内，走容器内部的service网络，结果还是没有解决
### cpu、内存
磁盘和网络都没有问题，怀疑可能是docker的cpu和内存的资源加载的不及时？这是之前的配置
```
      resources:
        limits:
          cpu: '4'
          memory: 4000Mi
        requests:
          cpu: 100m
          memory: 1000Mi
```
于是将request和limit调整一致，均为4核4G，结果还是没有解决
### mysql指标
连接数、锁表都很正常
### mysql性能分析
好吧，从mysql自身入手， 通过开启```show profiles```，观察下一个简单的查询语句是阻塞到哪里了，结果是sending data
> 当执行一个查询时，MySQL会按照查询计划逐步执行不同的阶段。当MySQL完成了读取数据的阶段（例如扫描表、使用索引等），并且需要将结果发送给客户端时，就会出现"Sending data"阶段

好吧，到这无果了，对比一下和迁移前的差距，问题可能还是在配置文件上，于是去逐个阅读配置文件背后的原理，终于找到了一条可疑的配置
```
innodb_thread_concurrency=4
```
这个配置大概意思是限制了innodb并发的线程为4，当有四个线程正在并发执行查询或者事务的时候，其他的sql想要执行，就会放到一个等待队列，等待前面的线程释放资源后，才会获得一个执行线程去执行，等待队列按照FIFO的策略来执行， 这一点和java的线程池倒是设计的一样
#### 之前为什么可以？
之前是使用的默认值，默认值为0
> 当innodb_thread_concurrency参数设置为0时，表示未启用并发度限制，InnoDB存储引擎会根据系统负载和资源情况自动评估并发度。
> 
> InnoDB会根据以下几个因素来评估并发度：
> 
> 硬件资源：InnoDB会检测系统的硬件资源情况，包括CPU核心数、内存大小和磁盘I/O能力。根据可用的资源情况，它会决定是否创建新的执行线程。
> 
> 系统负载：InnoDB会监视当前的系统负载，包括并发查询和事务数量。如果系统负载较低，InnoDB可能会创建更多的执行线程来并发执行查询和事务。反之，如果系统负载较高，InnoDB可能会推迟或拒绝创建新的执行线程，以避免资源争用和性能下降。
> 
> 动态调整：InnoDB会根据实时的负载情况和资源利用率进行动态调整。它会监控执行线程的活动情况，并根据需要增加或减少执行线程的数量，以适应系统负载的变化。
> 
> 通过这些评估和动态调整，InnoDB能够自动优化并发度，以最大程度地利用系统资源并提高性能。然而，由于每个系统的配置和负载模式都不同，最佳的并发度可能会因情况而异。因此，对于特定的系统，可能需要根据实际情况进行调整和优化。
#### 为什么设置为4
对mysql了解不足， 这个值最早大家的建议都是设置为cpu的核数
#### 应该怎么做？
依据```percona.com```的文档给出的合理值应该是活跃用户线程小于64的时候使用默认值， 大于64的时候设置为128，然后根据测试情况逐步下调或上涨
