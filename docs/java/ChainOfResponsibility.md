## 背景介绍
某功能设计需要经过很复杂一段逻辑的校验，且该逻辑校验可以拆分为各个细粒度的校验供于其他功能块组装使用
## 代码结构
```code
## 责任链抽象类
@Slf4j
@Component
public  class AbstractPartialRuleHandler {

    //下一个节点
    public AbstractPartialRuleHandler nextPrepareHandler;

    //初始化规则
    @Autowired
    public PartialRuleHandler partialRuleHandler;

    @PostConstruct
    public void init(){
      this.setNextPrepareHandler(partialRuleHandler);
    }

    public void  setNextPrepareHandler(AbstractPartialRuleHandler nextPrepareHandler){
        this.nextPrepareHandler = nextPrepareHandler;
    }

    //提供子类实现,具体的规则
    public  PartialRuleByResultVO getResultMassage(PartialRuleModel partialRuleModel) throws Exception {
        return null;
    };
}
```
```code
## 规则A
@Slf4j
@Component
public class PartialRuleHandler extends AbstractPartialRuleHandler {

    @Override
    public PartialRuleByResultVO getResultMassage(PartialRuleModel param) throws Exception {
        /**
         *业务代码A
         */
    }
}
## 规则B
@Slf4j
@Component
public class PartialRuleSaleModesHandler extends AbstractPartialRuleHandler {

    @Override
    public PartialRuleByResultVO getResultMassage(PartialRuleModel param) throws Exception {
        /**
         *业务代码B
         */
    }
}
......不再追加描述

```
## 具体使用
因职责链中具体执行者使用到spring的bean操作一些业务代码，所以要将自己注册到bean容器中，在组装时，需要从bean中拿到具体执行者的bean再组装
```code
    @Autowired
    private PartialRuleHandler ruleHandler;
    @Autowired
    private PartialRuleSaleModesHandler saleModesHandler;
    public void bussiness(){
        PartialRuleModel param = new PartialRuleModel();
        ruleHandler.setNextPrepareHandler(saleModesHandler);
        ruleHandler.getResultMassage(param);
    }
也可初始化组装好默认规则,在不同的类下使用不同的bean组合
```

