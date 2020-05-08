## 什么是队列
队列与栈非常类似，不过队列是一种先进先出(FIFO)的线性表，基本的操作方式为入队和出队
## 顺序队列和链式队列
和栈一样可以用数组或者链表来实现队列，数组实现的方式为顺序队列，链表实现的方式为链式队列
```code
//基于数组的实现方式
public class ArrayQueue {
  private String[] items;
  private int n = 0; //队列大小
  private int head = 0;//队列首元素位置
  private int tail = 0;//队列末元素位置

  public ArrayQueue(int capacity) {
    items = new String[capacity];
    n = capacity;
  }
  //末尾元素下标如果和队列大小相同的话，无法入队
  //元素排入末尾
  public boolean enqueue(String item) {
    if (tail == n) return false;
    items[tail] = item;
    ++tail;
    return true;
  }
  //出队时如果首元素位置和末尾元素位置一致，说明已经没有元素了
  //首元素位置向后移
  public String dequeue() {
    if (head == tail) return null;
    String ret = items[head];
    ++head;
    return ret;
  }
}
```
## 循环队列
//todo
## 阻塞队列和并发队列
//todo
