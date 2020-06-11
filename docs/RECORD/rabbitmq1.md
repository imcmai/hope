# 简介
springboot集成rabbitmq，并使用延时队列插件发送延时消息，服务器配置的rabbitmq需安装rabbitmq_delayed_message_exchange插件
## maven
```code
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
```
## yml
spring:
  rabbitmq:
    host: 192.168.1.186
    port: 5672
    username: jinhuitai
    password: jinhuitai
## code
```code
@Component
@Slf4j
public class RabbitMQUtil {
    @Autowired
    RabbitTemplate rabbitTemplate;
    //通知的延时交换机
    private final String informExchange = "inform_delay_exchange";
    //通知的延时路由
    private final String informRoutingKey = "inform_delay_routingKey";
    //通知的延时队列
    private final String informQueue="inform_delay_queue";
    SimpleDateFormat sf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS");
    @RabbitListener(bindings = @QueueBinding(value = @Queue(informQueue), exchange = @Exchange(value = informExchange, type = "topic", delayed = "true"), key = informRoutingKey))
    @RabbitHandler
    public void rabbitMQDelayReciver(InformMQEntry content, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws Exception {
        log.info("收到消息：[" + JSON.toJSONString(content) + "] 接收时间："+ sf.format(new Date()));
        channel.basicAck(tag, false);
    }

    /**
     * 最大延迟24天
     *
     * @param content
     * @param millisecond
     */
    public void send(InformMQEntry content, Integer millisecond) {
        log.info("投递消息：[" + JSON.toJSONString(content) + "] 投递时间："+ sf.format(new Date())+"延时时间"+millisecond);
        rabbitTemplate.convertAndSend(informExchange, informRoutingKey, content, message -> {
            message.getMessageProperties().setDelay(millisecond);
            return message;
        });
    }
}


```
