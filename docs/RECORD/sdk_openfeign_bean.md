## 概述
需要开发一个SDK，这个SDK里依赖了openfeign，同时注册了一个feign client，
当我使用另一个服务去依赖这个SDK的时候，去注入这个feign client却失败了
## 解决
1. 使用spring.factories去加载config
```code
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.test.NotificationConfig
```
2. config手动注册feign client
```code
@Configuration
@Import({FeignClientsConfiguration.class})
//@AutoConfigureOrder(100)
public class NotificationConfig {
    @Autowired
    private RequestInterceptor requestInterceptor;

    @Bean
    public NoticeAPI noticeAPI(Contract contract, Decoder decoder, Encoder encoder) {
        return Feign.builder()
                .contract(contract)
                .encoder(encoder)
                .decoder(decoder)
                .requestInterceptor(requestInterceptor)
                .target(NoticeAPI.class, "https://baidu.com");

    }

public interface NoticeAPI {

    /**
     * 短信发送
     *
     * @return
     */
    @PostMapping("/notice/sms")
    public RemoteResult sendSms(@RequestBody SMSDTO smsDTO);
}
```
## 备注
手动注册后，自定义的拦截器会失效，所以需要手动给Feign client配置RequestInterceptor