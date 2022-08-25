## 动态路由
从接口中拿到路由信息，刷新scg的路由，同时支持增量的增删改，RouteDefinitionRepository支持增量的添加和删除，所以选择RouteDefinitionRepository作为实现
### 实现
```code
@Component
public class RJRouteDefinitionRepository implements RouteDefinitionRepository {
    private final Map<String, RouteDefinition> routes = synchronizedMap(new LinkedHashMap<String, RouteDefinition>());
    private final Logger logger = LoggerFactory.getLogger(this.getClass());
    @Autowired
    private RJConfigProvider rjConfigProvider;
    @Autowired
    private ApplicationContext applicationContext;

    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return Flux.fromIterable(routes.values());
    }

    @Override
    public Mono<Void> save(Mono<RouteDefinition> route) {
        return route.flatMap(r -> {
            if (ObjectUtils.isEmpty(r.getId())) {
                return Mono.error(new IllegalArgumentException("id may not be empty"));
            }
            routes.put(r.getId(), r);
            return Mono.empty();
        });
    }

    @Override
    public Mono<Void> delete(Mono<String> routeId) {
        return routeId.flatMap(id -> {
            if (routes.containsKey(id)) {
                routes.remove(id);
                return Mono.empty();
            }
            return Mono.empty();
        });
    }

    public void routesChange() {
        Map<String, RouteDefinition> routeDefinitionMap = rjConfigProvider.getRouteDefinitions().stream().collect(Collectors.toMap(RouteDefinition::getId, Function.identity()));
        routes.clear();
        routes.putAll(routeDefinitionMap);
        this.applicationContext.publishEvent(new RefreshRoutesEvent(this));
        this.logger.info("路由配置刷新成功");
    }

    public void routeAdd(String content) {
        RouteDefinition route = JSON.parseObject(content, RouteDefinition.class);
        this.save(Mono.just(route)).subscribe();
        this.applicationContext.publishEvent(new RefreshRoutesEvent(this));
    }

    public void routeDelete(String content) {
        RouteDefinition route = JSON.parseObject(content, RouteDefinition.class);
        this.delete(Mono.just(route.getId())).subscribe();
        this.applicationContext.publishEvent(new RefreshRoutesEvent(this));
    }

    public void routeUpdate(String content) {
        routeDelete(content);
        routeAdd(content);
    }
```
## 自定义异常
修改gateway返回的错误页，改为restful风格的响应
### 实现
```code
    @Bean
    @Order(Ordered.HIGHEST_PRECEDENCE)
    public ErrorWebExceptionHandler errorWebExceptionHandler(
            ErrorAttributes errorAttributes) {
        RJGlobalErrorExceptionHandler exceptionHandler = new RJGlobalErrorExceptionHandler(
                errorAttributes, this.resourceProperties,
                this.serverProperties.getError(), this.applicationContext);
        exceptionHandler.setViewResolvers(this.viewResolvers);
        exceptionHandler.setMessageWriters(this.serverCodecConfigurer.getWriters());
        exceptionHandler.setMessageReaders(this.serverCodecConfigurer.getReaders());
        return exceptionHandler;
    }
```
```code
public class RJGlobalErrorExceptionHandler extends DefaultErrorWebExceptionHandler {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());
    private static final String GW_ERROR_PREFIX="GW_";
    public RJGlobalErrorExceptionHandler(ErrorAttributes errorAttributes,
                                         WebProperties.Resources resources,
                                         ErrorProperties errorProperties,
                                         ApplicationContext applicationContext) {
        super(errorAttributes, resources, errorProperties, applicationContext);
    }

    @Override
    public Mono<Void> handle(ServerWebExchange exchange, Throwable throwable) {

        return super.handle(exchange, throwable).then(Mono.fromRunnable(
                ()->{
                    if(throwable instanceof ResponseStatusException){
                        long currTime=System.currentTimeMillis();
                        RJGatewayLog.log(exchange,currTime,currTime,((ResponseStatusException)throwable).getStatus().value());
                    }else{
                        logger.error("RJGlobalErrorExceptionHandler",throwable);
                    }
                }
        ));
    }

    /**
     * 修改默认响应体为RJ格式
     * @param request
     * @return
     */
    @Override
    protected Mono<ServerResponse> renderErrorResponse(ServerRequest request) {
        Map<String, Object> error =this.getErrorAttributes(request,
                this.getErrorAttributeOptions(request, MediaType.ALL));
        ConvertToRJResponse(error);
        return ServerResponse.status(this.getHttpStatus(error)).contentType(MediaType.APPLICATION_JSON).body(BodyInserters.fromValue(error));
    }

    @Override
    protected Mono<ServerResponse> renderErrorView(ServerRequest request) {
        return this.renderErrorResponse(request);
    }

    private void ConvertToRJResponse(Map<String, Object> error){
        error.put("err",GW_ERROR_PREFIX+String.valueOf(error.get("error")));
        error.put("data",null);
        error.remove("timestamp");
        error.remove("path");
        error.remove("requestId");
        error.remove("error");
        error.remove("message");
    }
```


或者修改gateway的错误页，改为自己自定义的错误页
```code
    @Override
    protected Mono<ServerResponse> renderErrorView(ServerRequest request) {
        Map<String, Object> error =this.getErrorAttributes(request,
                this.getErrorAttributeOptions(request, MediaType.ALL));
        if(error.get("status").equals(404)){
            return ServerResponse.status(this.getHttpStatus(error)).contentType(MediaType.TEXT_HTML).body(BodyInserters.fromValue(ErrorHTML.NOT_FOUND_HTML));
        }
        if(error.get("status").equals(503)){
            return ServerResponse.status(this.getHttpStatus(error)).contentType(MediaType.TEXT_HTML).body(BodyInserters.fromValue(ErrorHTML.SERVICE_UNAVAILABLE));
        }
        return this.renderErrorResponse(request);
    }
```
