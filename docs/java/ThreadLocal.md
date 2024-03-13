## Threadlocal是什么
我们先看一下Threadlocal的注释

```code
/**
 * This class provides thread-local variables.  These variables differ from
 * their normal counterparts in that each thread that accesses one (via its
 * {@code get} or {@code set} method) has its own, independently initialized
 * copy of the variable.  {@code ThreadLocal} instances are typically private
 * static fields in classes that wish to associate state with a thread (e.g.,
 * a user ID or Transaction ID).
 *
 * <p>For example, the class below generates unique identifiers local to each
 * thread.
 * A thread's id is assigned the first time it invokes {@code ThreadId.get()}
 * and remains unchanged on subsequent calls.
 * <pre>
 * import java.util.concurrent.atomic.AtomicInteger;
 *
 * public class ThreadId {
 *     // Atomic integer containing the next thread ID to be assigned
 *     private static final AtomicInteger nextId = new AtomicInteger(0);
 *
 *     // Thread local variable containing each thread's ID
 *     private static final ThreadLocal&lt;Integer&gt; threadId =
 *         new ThreadLocal&lt;Integer&gt;() {
 *             &#64;Override protected Integer initialValue() {
 *                 return nextId.getAndIncrement();
 *         }
 *     };
 *
 *     // Returns the current thread's unique ID, assigning it if necessary
 *     public static int get() {
 *         return threadId.get();
 *     }
 * }
 * </pre>
 * <p>Each thread holds an implicit reference to its copy of a thread-local
 * variable as long as the thread is alive and the {@code ThreadLocal}
 * instance is accessible; after a thread goes away, all of its copies of
 * thread-local instances are subject to garbage collection (unless other
 * references to these copies exist).
 *
 * @author  Josh Bloch and Doug Lea
 * @since   1.2
 */
```
得出以下结论

- 提供线程局部变量，每个线程都有自己的变量副本
- 只要线程处于活动状态且Threadlocal实例可访问，每个线程都拥有对其线程局部变量副本的弱引用,不会被垃圾回收掉
- 线程消失后，线程局部实例的所有副本都将进行垃圾回收（除非存在对这些副本的其他引用)
## 源码阅读
### 首先看一下get的源码
```code
  /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
### get执行顺序
1. 获取当前线程内部的threadlocalmap
2. map存在则去获取value
3. map不存在去调用setInitialvalue
### getMap(t)的实现
```code
  ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
返回的当前线程的一个threadlocals变量，我们继续来看这个threadlocals

```code
  /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```
表明这是一个由threadlocal类来维护的与此线程有关的threadlocal值,threadlocalmap的实现是一个内部类Entry继承弱引用，以threadlocal为key的map
### getEntry的实现


//todo 完善后续getEntry set setInitialvalue