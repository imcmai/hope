## 背景
在spring cloud的体系下，需要对所有请求做referer认证，防止CSRF，放在各个模块的话，比较冗余，所以直接放在了gateway下
### 通过gateway网关对请求做referer认证
```code
public class RefererGlobalFilter implements GlobalFilter, Ordered {
    @Autowired
    RefererProperties properties;
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        ServerHttpResponse response = exchange.getResponse();
        String referer = request.getHeaders().getFirst("referer");
        if (referer == null) {
            response.setStatusCode(HttpStatus.FORBIDDEN);
            return exchange.getResponse().setComplete();
        }
        java.net.URL url = null;
        try {
            url = new java.net.URL(referer);
        } catch (MalformedURLException e) {
            response.setStatusCode(HttpStatus.FORBIDDEN);
            return exchange.getResponse().setComplete();
        }
        if(properties.getRefererDomain().contains(url.getHost())){
            return chain.filter(exchange);
        }else {
            response.setStatusCode(HttpStatus.FORBIDDEN);
            return exchange.getResponse().setComplete();
        }
    }

    @Override
    public int getOrder() {
        return 1;
    }
}
```
配置类
```code
package com.jinhuitai.oacloud.gateway.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.util.HashSet;
import java.util.List;
import java.util.Set;

@Component
@ConfigurationProperties(prefix = "referer")
public class RefererProperties {
    private Set<String> refererDomain = new HashSet<>();

    public Set<String> getRefererDomain() {
        return refererDomain;
    }

    public void setRefererDomain(Set<String> refererDomain) {
        this.refererDomain = refererDomain;
    }
}

```
application.yml
```code
referer:
  refererDomain:
    - 192.168.1.105
    - 192.168.1.101
```