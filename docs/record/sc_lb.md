## 背景描述
两地域部署K8S集群，每个集群部署的都有网关的实例，请求通过ingrees路由到Spring cloud gateway后，gateway需要实现例如北京的请求转发到北京的实例，减少网络损耗
## 客户端改造
```code
spring:
  cloud:
    nacos:
      discovery:
        metadata:
          rj.zone: ${rj.zone}
```
追加配置，通过启动参数获取，部署该实例的时候指定实例所在的区域
## 网关改造
```code

/**
 * @author zrh
 * @description LB策略自动装配
 * @date 2022/4/11 17:29
 */
@Configuration(proxyBeanMethods = false)
@LoadBalancerClients(defaultConfiguration = RJLoadBalancerConfig.class)
public class RJLoadBalancerAutoConfig {
}

```
```code

/**
 * @author zrh
 * @description 生成LB策略Bean
 * @date 2022/4/11 16:21
 */
@Configuration(proxyBeanMethods = false)
public class RJLoadBalancerConfig {
    private final String ZONE_KEY = "rj.lb.zone";
    @Bean
    public ReactorServiceInstanceLoadBalancer rjLoadBalance(LoadBalancerClientFactory loadBalancerClientFactory, Environment environment) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        String gatewayZone = environment.getProperty(ZONE_KEY);
        return new RJLoadBalancer(loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),name,gatewayZone);
    }
}

```
网关也通过环境变量指定了当前所在的区域
```code

/**
 * @author zrh
 * @description 区域负载均衡策略类，每个服务单独持有一个实例
 * @date 2022/4/11 15:00
 */
public class RJLoadBalancer implements ReactorServiceInstanceLoadBalancer {
    private final Logger log = LoggerFactory.getLogger(RJLoadBalancer.class);
    //网关所在区域
    private final String gatewayZone;
    //转发服务的名称
    private final String serviceId;
    //轮询随机数
    private final AtomicInteger position;
    //区域轮询随机数
    private final AtomicInteger zonePosition;
    private final String CLIENT_ZONE_KEY = "rj-zone";
    private ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;

    public RJLoadBalancer(ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider, String serviceId, String gatewayZone) {
        this(serviceInstanceListSupplierProvider, serviceId, gatewayZone, new Random().nextInt(1000), new Random().nextInt(1000));
    }

    public RJLoadBalancer(ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider, String serviceId, String gatewayZone, int seedPosition, int zoneSeedPosition) {
        this.serviceInstanceListSupplierProvider = serviceInstanceListSupplierProvider;
        this.gatewayZone = gatewayZone;
        this.serviceId = serviceId;
        this.position = new AtomicInteger(seedPosition);
        this.zonePosition = new AtomicInteger(zoneSeedPosition);
    }

    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier = serviceInstanceListSupplierProvider
                .getIfAvailable(NoopServiceInstanceListSupplier::new);
        return supplier.get(request).next()
                .map(serviceInstances -> processInstanceResponse(supplier, serviceInstances));
    }

    private Response<ServiceInstance> processInstanceResponse(ServiceInstanceListSupplier supplier, List<ServiceInstance> serviceInstances) {
        Response<ServiceInstance> serviceInstanceResponse = this.getInstanceResponse(serviceInstances);
        if (supplier instanceof SelectedInstanceCallback && serviceInstanceResponse.hasServer()) {
            ((SelectedInstanceCallback) supplier).selectedServiceInstance(serviceInstanceResponse.getServer());
        }
        return serviceInstanceResponse;
    }

    private Response<ServiceInstance> getInstanceResponse(List<ServiceInstance> instances) {
        if (instances.isEmpty()) {
            if (log.isWarnEnabled()) {
                log.warn("Gateway No servers available:" + serviceId);
            }
            return new EmptyResponse();
        } else {
            List<ServiceInstance> zoneHitService = instances.parallelStream().filter((service) -> {
                String zone = service.getMetadata().get(CLIENT_ZONE_KEY);
                return gatewayZone.equals(zone);
            }).collect(Collectors.toList());
            if (zoneHitService.isEmpty()) {
                return roundRobin(position, instances);
            } else {
                return roundRobin(zonePosition, zoneHitService);
            }
        }
    }

    private Response<ServiceInstance> roundRobin(AtomicInteger position, List<ServiceInstance> instances) {
        int pos = Math.abs(position.incrementAndGet());
        ServiceInstance serviceInstance = instances.get(pos % instances.size());
        if (serviceInstance.getPort() == 443) {
            NacosServiceInstance instance = (NacosServiceInstance) serviceInstance;
            instance.setSecure(true);
            return new DefaultResponse(instance);
        }
        log.info("Gateway load balancer hit:" + JSON.toJSONString(serviceInstance));
        return new DefaultResponse(serviceInstance);
    }
}


```
