## 问题
服务端出现java.IO.Exception:Broken pipe , 从网关找不到该请求的日志，网关是由nginx来负载均衡的，在nginx中有日志，日志如下
```code
172.17.***.** - - [13/May/2022:14:37:22 +0800] "GET /api/*/**/**/***/** HTTP/1.1" 499 0 "-" "Java/1.8.0_271"
```
网关记录日志代码如下
```code
public class RJLogGlobalFilter implements GlobalFilter, Ordered {
    private final Logger logger = LoggerFactory.getLogger(RJGatewayRequestFilter.RJGATEWAYREQUESTLOGGERNAME);
    public RJLogGlobalFilter() {    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long startTime=System.currentTimeMillis();
        return chain.filter(exchange)
                .doOnError(throwable->{
                    Mono.fromRunnable(()->{
                        RJGatewayLog.log(exchange,startTime,System.currentTimeMillis(),((ResponseStatusException)throwable).getStatus().value());
                    });
                })
                .then(Mono.fromRunnable(()->{
                    long endTime=System.currentTimeMillis();
                    RJGatewayLog.log(exchange,startTime,endTime,0);
                }));
    }

    @Override
    public int getOrder() {
        return NettyWriteResponseFilter.WRITE_RESPONSE_FILTER_ORDER-1;
    }
}

```
## 复现
本地启动一个服务端，由网关代理，服务端代码为线程休眠10S，客户端发起请求，在请求未结束时关闭该进程， 此时问题复现
## 解决
```code
return chain.filter(exchange)
        .doOnError(throwable->{
            Mono.fromRunnable(()->{
                RJGatewayLog.log(exchange,startTime,System.currentTimeMillis(),((ResponseStatusException)throwable).getStatus().value());
            });
        })
        .doOnCancel(()->{
            long endTime=System.currentTimeMillis();
            RJGatewayLog.log(exchange,startTime,endTime,499);
        })
        .then(Mono.fromRunnable(()->{
            long endTime=System.currentTimeMillis();
            RJGatewayLog.log(exchange,startTime,endTime,0);
        }));
```
在客户端撤销连接时，记录日志
