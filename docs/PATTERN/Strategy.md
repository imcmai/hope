## 背景介绍
权限的资源划分为了多种类型， 如菜单，按钮，字段，流程(实际权限粒度控制到了记录级别)等等...，前端在特定场景下，需要根据权限类型和权限id查询权限名称，格式如下
```code
{
   "menu":[1,2,3],
   "field":[1,2,3],
   "process":[1,2,3]
   .....
   ...
}
```
改造之前 后端的伪代码为
```code
    public List<String> listUserPermissionsName(Param param) {
        //对应上述的前端数据结构
        Map<String,List<String>> permissionIds = param.getParam();
        List<String> result = new ArrayList<>();
        for (Map.Entry<String, List<String>> entry : permissionIds.entrySet()) {
            PermissionTypeEnum typeEnum = PermissionTypeEnum.parseFromName(entry.getKey());
            if(typeEnum==PermissionTypeEnum.MENU){
                //查询权限...
                result.addAll(menuNames);
            }
            if(typeEnum==PermissionTypeEnum.FIELD){
                //查询权限...
                result.addAll(menuNames);
            }
            ........
            ........
        }
        return result;
    }
```
### 总结
因为权限的粒度其实已经控制到了记录级(少量记录的数据，比如工作流的流程，表单等等)
所以后续可能会有更多的扩展和更多的分支，每个分支的逻辑可能会越来越复杂，遂在闲暇之余对代码做了改造
### 改造后代码结构
```code
//权限策略
public interface PermissionStrategy {
    /**
     * 装载权限名称
     * @param entry
     * @return
     */
    List<String> payloadPermissionName(Map.Entry<String, List<String>> entry);
}
```
```code
//菜单策略类
@Component
public class PermissionMenu implements PermissionStrategy, InitializingBean {
    @Autowired
    MenuService menuService;
    @Override
    public List<String> payloadPermissionName(Map.Entry<String, List<String>> entry) {
        //处理逻辑
        return menuNames;
    }
    //注册到工厂
    @Override
    public void afterPropertiesSet() throws Exception {
        PermissionFactory.register(PermissionTypeEnum.MENU,this);
    }
}
//流程策略类
@Component
public class PermissionProcess implements PermissionStrategy, InitializingBean {
    @Autowired
    private ProcessFeignService processFeignService;
    @Autowired
    UserPermissionService userPermissionService;

    @Override
    public List<String> payloadPermissionName(Map.Entry<String, List<String>> entry) {
        //处理逻辑
        return processNames;
    }
    //注册到工厂
    @Override
    public void afterPropertiesSet() throws Exception {
        PermissionFactory.register(PermissionTypeEnum.PROCESS, this);
    }
}

//其他就不一一写了
........
```
```code
//权限工厂
public class PermissionFactory {
    private static Map<PermissionTypeEnum, PermissionStrategy> permissionStrategyMap = new ConcurrentHashMap<>();
    public static PermissionStrategy getPermissionStrategy(PermissionTypeEnum typeEnum) {
        return permissionStrategyMap.get(typeEnum);
    }
    public static void register(PermissionTypeEnum typeEnum,PermissionStrategy permissionStrategy){
        permissionStrategyMap.put(typeEnum,permissionStrategy);
    }
}

```code
```code
//改造后调用方代码
    public List<String> listUserPermissionsName(Param param) {
        Map<String,List<String>> permissionIds = param.getParam();
        List<String> result = new ArrayList<>();
        for (Map.Entry<String, List<String>> entry : permissionIds.entrySet()) {
            PermissionTypeEnum typeEnum = PermissionTypeEnum.parseFromName(entry.getKey());
            PermissionStrategy permissionStrategy = PermissionFactory.getPermissionStrategy(typeEnum);
            List<String> permissionNames =permissionStrategy.payloadPermissionName(entry,typeEnum);
            result.addAll(permissionNames);
        }
        return result;
    }
```
### 感悟
设计模式和算法普遍已经被各种框架给封装，实际应用中很少有场景可以用到，闲暇之余可以多思考一下改善代码