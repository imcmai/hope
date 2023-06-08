## 说明
飞书有邮件频率限制， 每个账号大概是100s 200封的样子，  为了提高邮件的发送频率，所以需要用到多个发信帐号，
## 思路
1. 线程池来接收邮件请求,此处保证邮件接收服务能最大限度的接收邮件请求
2. 线程在执行时使用令牌池来控制邮件频率，此处保证最大限度的利用飞书邮件的发送频率， Thread.sleep也是可以的，只是不够精确
## 部分代码代码实现
### 线程池
```
public class MailExecutorService {
    private static final Logger logger = LoggerFactory.getLogger(MailExecutorService.class);
    private static ExecutorService es = new ThreadPoolExecutor(6,
            12, 120L, TimeUnit.SECONDS,
            new ArrayBlockingQueue<Runnable>(2000),
            new ThreadFactoryBuilder().setUncaughtExceptionHandler((t, ex) -> {
                logger.info("线程:{},异常:{}", t, ex);
                ex.printStackTrace();
            }).setNameFormat("notice-mail-pool-%d").build(),
            new ThreadPoolExecutor.AbortPolicy());
    public static ExecutorService getEs() {
        return es;
    }
}
```
### 令牌池
```code
    private static final int MAIL_LIMITER_NUM = 5;
    private static final List<RateLimiter> MAIL_LIMITERS = new ArrayList<>();
    private static final Map<RateLimiter, JavaMailSenderImpl> MAIL_SENDERS = new ConcurrentHashMap<>();
    private static final AtomicInteger POSITION = new AtomicInteger(new Random().nextInt(1000));

    static {
        //账号和限流器初始化
    }
    /**
     * 1.尝试从限流器中直接获得令牌，拿到令牌后返回对应的mail sender
     * 2.如果没有从限流器中直接拿到令牌，按照轮询的方式，挑选一个限流器阻塞获取令牌
     *
     * @return JavaMailSenderImpl
     */
    public static JavaMailSenderImpl getSender() {
        for (int i = 0; i < MAIL_LIMITERS.size(); i++) {
            RateLimiter mailLimiter = MAIL_LIMITERS.get(i);
            if (mailLimiter.tryAcquire()) {
                return MAIL_SENDERS.get(mailLimiter);
            }
        }
        int pos = Math.abs(POSITION.incrementAndGet());
        RateLimiter roundRobinLimiter = MAIL_LIMITERS.get(pos % MAIL_LIMITER_NUM);
        roundRobinLimiter.acquire();
        return MAIL_SENDERS.get(roundRobinLimiter);
    }
```
### 入口代码
```code
            ExecutorService es = MailExecutorService.getEs();
            es.execute(() -> {
                JavaMailSenderImpl mailSender = MailSendFactory.getSender();
                //发送邮件
            });
```
