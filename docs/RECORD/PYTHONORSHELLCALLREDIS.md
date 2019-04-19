## python调用redis
```code
import redis

redisConnectPool = redis.ConnectionPool(host='*', port=6379)  # redis连接池

def callRedis():
    try:
        redisConnect = redis.Redis(connection_pool=redisConnectPool)
        runDitingScriptCount = None
        runDitingScriptCount = redisConnect.get('runDitingScriptCount')
        if (runDitingScriptCount is None):
            runDitingScriptCount = '0'
        log.logger.info("当前在执行diting脚本次数:%s", runDitingScriptCount);
        if (int(runDitingScriptCount) < 10):
            redisConnect.set('runDitingScriptCount', int(runDitingScriptCount) + 1)
            return True
        else:
            return False
    except Exception, e:
        log.logger.error(str(e))
        return True
        
```
## shell调用redis,并且接收结果集做运算回写到redis
```code
#!/usr/bin/env bash
: ${username=`redis-cli -h  * -p 6379 -a 123456 get 'test'`}  --变量接收get的值
declare -i username   --定义变量为int类型
username=username+1  --变量++
redis-cli -h  *
-p 6379 -a 123456 set 'test' $username --把变量值重新set 
echo $username   
```
