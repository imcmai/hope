## 代码示例
```code
public class Account {
    private Integer salary;
    public Integer getSalary() {
        return salary;
    }

    public void setSalary(Integer salary) {
        synchronized (salary) {
            try {
                Thread.currentThread().sleep(10000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            this.salary = salary;
        }
    }
}

public class AccountCopy {
    private Integer salary;

    public Integer getSalary() {
        return salary;
    }

    public void setSalary(Integer salary) {
        synchronized (salary) {
            this.salary = salary;
        }
    }
}

public class Main {

    public static void main(String[] args) {
        Account account = new Account();
        AccountCopy accountCopy = new AccountCopy();
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    account.setSalary(1);
                    System.out.println(account.getSalary()+"t1");
                }
            }
        });
        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    accountCopy.setSalary(1);
                    System.out.println(accountCopy.getSalary()+"t2");
                }
            }
        });
        t1.start();
        t2.start();
    }
}
```
## 代码执行结果
在t1没有执行完之前，t2被阻塞
## 原因
Integer缓存128以下的数字，内存地址为相同内存地址，sync锁的为同一个对象，同理
String也有一样的问题,锁对象会被重用
## 总结
Integer和String不适合作为锁对象，锁对象应该具有私有、不可变的特性，非私有会导致锁粒度提高，可变会导致锁失效