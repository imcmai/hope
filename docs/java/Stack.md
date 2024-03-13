## 什么是栈
栈是一种后进先出(LIFO)的受限线性表，只能从一端插入和提取数据
## 为什么需要栈
前面说了他是一个受限的线性表，所有操作用数组或者链表均可实现，但是就是因为栈受限，满足一些特定场合的抽象，减少开发人员关注的层次。
## 栈的应用场景
经典的场景，例如浏览器的前进和后退功能
## 数组实现栈
```code
public class ArrayStack {
  private String[] items;  // 数组
  private int count;       // 栈中元素个数
  private int n;           //栈的大小
  //初始化栈空间
  public ArrayStack(int n) {
    this.items = new String[n];
    this.n = n;
    this.count = 0;
  }

  public boolean push(String item) {
    if (count == n) return false;
    items[count] = item;
    ++count;
    return true;
  }
  
  public String pop() {
    if (count == 0) return null;
    String tmp = items[count-1];
    --count;
    return tmp;
  }
}
```
## 链表实现栈
//todo
## 栈的时间空间复杂度
出入栈均为O(1)