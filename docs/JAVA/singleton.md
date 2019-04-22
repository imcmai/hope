## java中单例模式的定义
一个类有且仅有一个实例，并且自行实例化向整个系统提供
## 单例解决的问题
控制实例数目，节省系统资源
### 单例带来的优势
1. 灵活，由类来控制对象的实例过程
2. 实例控制，确保所有的类访问的都是唯一实例
3. 减少开销
### 单例的缺点
1. 违反面向对象的单一职责
2. 不适于扩展
## 单例最佳实践
这是之前做的一道笔试题
> 1：现在有Message和MessageFactory两个接口如下：
  public interface Message extends Serializable {
       void printMessage();
  }
  public interface MessageFactory {
      Message newMessage(String countryCode);
  }
  请编写一个实现MessageFactory 接口的类，并使用到单例模式。
  
```code
package subject.demo.one;
/**
 * @author cmai
 * @createDate 2018/11/12
 * @description message factory double check singleton impl
 */
public class MessageFactoryImpl implements MessageFactory {
    private volatile static Message singleton;
    @Override
    public Message newMessage(String countryCode) {
        if (singleton == null) {
            synchronized (MessageFactoryImpl.class) {
                if (singleton == null) {
                    singleton = () -> {
                        System.out.println(countryCode);
                    };
                }
            }
        }
        return singleton;
    }

    public static void main(String[] args) {
        MessageFactoryImpl messageFactory = new MessageFactoryImpl();
        Message message = messageFactory.newMessage("test");
        message.printMessage();
    }
}
```
我们来解读一下
1. 双检查是为了在对象创建成功的情况下不必要加锁，所以在外层直接返回实例对象
2. volatile的作用是为了避免jvm对代码进行重排序
## volatile重排带来的问题
创建对象我们认为的执行顺序应该是
1. 分配一块内存M
2. 内存M初始化单例对象
3. 将实例的指针指向M
在创建对象的时候经过jvm的优化可能就会变成
1. 分配一块内存M
2. 将实例的指针指向M
3. 内存M初始化单例对象
这个时候就有可能触发空指针问题(假如发生线程切换)，这就是volatile的作用，禁止指令重排序
## 其他
volatile在JDK1.5之后，就加入了happen-before规则之中，后续的juc并发包，可见性基本都依赖于volatile
