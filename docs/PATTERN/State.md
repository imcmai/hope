## 状态模式描述
当一个对象状态发生改变时，可以改变其行为
## 状态模式实践
```code
//抽象状态接口
public abstract class State {
    Context context;
    public void setContext(Context context) {
        this.context = context;
    }
    public abstract void handleA();
    public abstract void handleB();
}
//状态A
public class ConcreteStateA extends State {
    @Override
    public void handleA() {}  //当前状态处理逻辑
​
    @Override
    public void handleB() {
        super.context.setCurrentState(Context.contreteStateB);  //切换到状态B        
        super.context.handleB();  //执行状态B的任务
    }
}
//状态B
public class ConcreteStateB extends State {
    @Override
    public void handleB() {}  //当前状态处理逻辑
  
    @Override
    public void handleA() {
        super.context.setCurrentState(Context.contreteStateA);  //切换到状态A
        super.context.handleA();  //执行状态A的任务
    }
}
//状态管理
public class Context {
    public final static ConcreteStateA contreteStateA = new ConcreteStateA();
    public final static ConcreteStateB contreteStateB = new ConcreteStateB();
​
    private State CurrentState;
    public State getCurrentState() {return CurrentState;}
​
    public void setCurrentState(State currentState) {
        this.CurrentState = currentState;
        this.CurrentState.setContext(this);
    }
​
    public void handleA() {this.CurrentState.handleA();}
    public void handleB() {this.CurrentState.handleB();}
}
public class client {
    public static void main(String[] args) {
        Context context = new Context();
        context.setCurrentState(new ContreteStateA());
        context.handleA();
        context.handleB();
    }
}
```
## 总结
和策略模式有相通之处，但是策略模式是只选择一个策略适用于代码中，状态模式则是组织多个状态形成状态的转换图，不止策略模式和状态模式，很多设计模式其实都很相近
,另外设计模式具体结合到业务的时候，要根据具体的业务灵活运用