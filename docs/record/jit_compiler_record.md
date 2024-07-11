## 代码产生的问题
先贴如下代码
``` code
    //代码A
    public static void main(String[] args) throws InterruptedException {
        Test a = new Test();
        a.start();
        for (; ; ) {
            if (a.isFlag()) {
                System.out.println("1");
            }
        }
    }

    static class Test extends Thread {

        private boolean flag = false;

        public boolean isFlag() {
            return flag;
        }

        @Override
        public void run() {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            flag = true;
            System.out.println(flag);

        }
}
```
代码执行流程
1. 主线程运行，启动了a线程
2. a线程sleep，主线程开始循环
3. a线程修改flag值  

我们预期的结果是a线程修改flag值后，死循环内可以拿到flag最新的值，最终输出"1"。
实际的运行结果确实if判断始终进不去，我们可以通过volatile来标记flag变量，让main线程每次读取都从主存中获取最新值，但是我想搞明白的却是，如果没有volatile，难道主存和线程副本内存就不交互了吗？
### 补充知识
jmm规定了主存和线程工作内存，工作内存是逻辑概念，实际可能指的是cpu缓存，线程会在自己的工作内存中操作，如果有修改值，会写入主存，然后根据cpu缓存一致性协议，其他用到该变量的线程，会标记此变量为无效，重新从主存中拉取最新的值
### 产生的疑惑
代码执行结果和缓存一致性协议产生了冲突
### 补充代码
在还没有解决上面的疑惑的时候，发现了另一个问题,关键代码改成如下，死循环中main线程就可以读到最新的flag值了
```code
    //代码B
    public static void main(String[] args) throws InterruptedException {
        Test a = new Test();
        a.start();
        for (; ; ) {
            if (a.isFlag()) {
                System.out.println("1");
            }else{
                System.out.println("2");
            }
        }
    }
```
## 关于这些问题得到的结果
代码A在启动时加上-Xint参数，发现执行结果是正确的，可以正常读到修改后的值，-Xint会让jvm通过字节码解释执行，而不是通过JIT优化后执行，所以问题可以定位为JIT优化了我的代码， 查阅相关资料后，JIT对于代码A可能会优化为
```code
        boolean isFlag = a.isFlag();
        for (; ; ) {
            if (isFlag) {
                System.out.println("1");
            }
        }
```
也就是取值的这个操作被抽离到了循环外，导致出现了这样的问题，我们可以做个验证,代码A改为延迟1s,可以正常进入if分支
```code
    //代码C
    public static void main(String[] args) throws InterruptedException {
        Test a = new Test();
        a.start();
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        for (; ; ) {
            if (a.isFlag()) {
                System.out.println("1");
            }
        }
    }
```
这样，是可以佐证我们上面的结论，当然最好还是能够查看JIT优化后的代码。
代码B，加上了else为何就可以正常执行，其实问题不在else，问题在println中用到sync，JIT就不会对这个循环做任何冒进的优化了。
```code
    public void println(String x) {
        synchronized (this) {
            print(x);
            newLine();
        }
    }
```
## 总结
JIT优化器对于只要优化后不违背happens-before规则的代码，是可以任意优化的，由此会产生出一些不可预知的问题。。
## 相关资料
1. 我在v2ex的提问(https://www.v2ex.com/t/671776#reply42)
2. stackoverflow的回答(https://stackoverflow.com/questions/25425130/loop-doesnt-see-value-changed-by-other-thread-without-a-print-statement)
3. RednaxelaFX在知乎的回答(https://www.zhihu.com/question/39458585/answer/81521474)

