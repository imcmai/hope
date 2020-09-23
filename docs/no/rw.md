<!-- 
# 不合理的代码
## 通过类对象引用类静态变量
```code
/**
 * 系统日志：切面处理类
 */
@Slf4j
@Aspect
@Component
public class SysLogAspect {
    @Autowired
    SysRequestLogService sysRequestLogService;
    //在注解的位置切入代码
    @Pointcut(value="@annotation(AnnotationLog)")
    public void logPoinCut() {
    }
    //切面 配置通知
    @Before(value="logPoinCut()", argNames = "joinPoint")
    public void saveSysLog(JoinPoint joinPoint) {
        //保存日志
        SysRequestLog operlog = new SysRequestLog();
        try {
            //生成主键
            IdWorker worker2 = new IdWorker(1);
        }
    }
}

public class IdWorker {

    private final long workerId;
    private final static long twepoch = 1288834974657L;
    private long sequence = 0L;
    private final static long workerIdBits = 4L;
    public final static long maxWorkerId = -1L ^ -1L << workerIdBits;
    private final static long sequenceBits = 10L;
    private final static long workerIdShift = sequenceBits;
    private final static long timestampLeftShift = sequenceBits + workerIdBits;
    public final static long sequenceMask = -1L ^ -1L << sequenceBits;
    private long lastTimestamp = -1L;
    public IdWorker(final long workerId) {
        super();
        if (workerId > this.maxWorkerId || workerId < 0) {







            throw new IllegalArgumentException(String.format(
                    "worker Id can't be greater than %d or less than 0",
                    this.maxWorkerId));
        }
        this.workerId = workerId;
    }
}
```
## 应以常量作为equals的调用方，如果对象为空调用equals会触发空指针
```code
@Slf4j
@Component
public class PostContractERPService implements SimpleJob{
    @Override
    public void execute(ShardingContext shardingContext) {
        try {
            if(returnFlag.equals("F")){
                //同步post信息失败，发送通知邮件
            }
        } catch (Exception e) {
            log.error("抛单信息同步到ECP失败,postmain信息:{}", JSON.toJSONString(postMain));
        }
    }
}
``` 
## 线程池参数设置,最好自定义线程名字，方便排查问题，同样根据业务来看阻塞队列过大，最大线程可能一直无法发挥用处
```code
@Slf4j
@RestController
@RequestMapping("/contractInfo")
public class ContractInfoController {
    ThreadPoolExecutor executor = new ThreadPoolExecutor(2,5,30, TimeUnit.MINUTES,new ArrayBlockingQueue(200));
```
## 常量名全部大写，而且最好能表达常量的含义，不需要驼峰，也不需要刻意短的命名
```code
public class ContractAPPCode extends AbstracAppCode {
    public static final ContractAPPCode ERROR_EmptyParameter = new ContractAPPCode("参数为空!","7100002");
    public static final ContractAPPCode ERROR_PARAM_AGENT_NAME = new ContractAPPCode("代理商名称为空!","7100003");
    public static final ContractAPPCode ERROR_PARAM_CONTRACTASSIGN_ERR = new ContractAPPCode("合同金额分配异常！","7100004");
    public static final ContractAPPCode ERROR_PARAM_ASSIGN_ERR = new ContractAPPCode("生成收款单异常！","7100005");
    public static final ContractAPPCode ERROR_USERId = new ContractAPPCode("操作人信息为空！","7100006");
    public static final ContractAPPCode ERROR_Comparative_Amount = new ContractAPPCode("申请退款金额不能超过可用余额！","7100007");
    public static final ContractAPPCode ERROR_ISCurrentApprover = new ContractAPPCode("您不是当前审批人!","7100008");

    public static final ContractAPPCode ERROR_CONTRACT_NOTEXIS = new ContractAPPCode("合同中心不存在，该合同信息，请联系开发人员！","7100009");
    public static final ContractAPPCode ERROR_CONTRACT_NOTDiscard = new ContractAPPCode("合同中心执行更新废弃状态失败！","7100010");
    public static final ContractAPPCode ERROR_CONTRACT_EXISDiscard = new ContractAPPCode("该合同已经被废弃过！","7100011");

```
## 常量最好不要直接出现在代码里
```code
//针对 总条数 大于 1000000 目前忽略
            if (jsonObject.get("status").equals(200) && count > 0
                    && !Strings.isNullOrEmpty(jsonObject.get("data").toString())

```
## 取反逻辑不利于快速理解代码,所有!可以做到的，不用!一样可以做到
```code
        if(!(i1>0)){
            String msg = "待办合同款项分配-插入分配记录失败！";
            throw new ContractException(ContractAPPCode.ERROR_PARAM_CONTRACTASSIGN_ERR.getAppCode(),msg);
        }
```
# 个人想法

# 如何提高代码质量
## 不要轻易复制逻辑代码
避免有的地方有改动，要同步到其他的地方，如果有遗漏，就会产生问题，最好重构它
## 排期内最好预留时间对所写代码重构设计
一般为五分之一的时间，好的代码设计不仅不不会浪费这五分之一的时间，还将对以后产生深远的影响，重点在提高可维护性和可读性
## 代码量
有足够多的代码量才会对各种各样的规范有自己的理解，总结，反思
..........
.........
....... -->
