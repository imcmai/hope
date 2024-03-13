## ReentrantLock在dubbo中的应用
dubbo版本:dubbox-2.8.4
## 思考
为何tcp协议本身是异步的，而dubbo的rpc用到tcp来实现，为什么dubbo调用服务是同步的
## 关键源码
```code
public class DubboInvoker<T> extends AbstractInvoker<T> {
    protected Result doInvoke(final Invocation invocation){
        return (Result) currentClient.request(inv, timeout).get();
    }
```

```code
public Object get(){
    return get(timeout);
}
public Object get(int timeout){
    if (timeout <= 0) {
        timeout = Constants.DEFAULT_TIMEOUT;
    }
    if (! isDone()) {
        long start = System.currentTimeMillis();
        lock.lock();
        try {
            while (! isDone()) {
                done.await(timeout, TimeUnit.MILLISECONDS);
                if (isDone() || System.currentTimeMillis() - start > timeout) {
                    break;
                }
            }
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            lock.unlock();
        }
        if (! isDone()) {
            throw new TimeoutException(sent > 0, channel, getTimeoutMessage(false));
        }
    }
    return returnFromResponse();
}
```
## 结论
dubbo在调用服务时利用lock锁，阻塞了调用线程，直到rpc返回结果