## J.U.C包下阻塞队列
```java
   /** 当前队列条目 **/
    private int count = 0;
    /** 实际存储的数据 **/
    private Object[] items;
    final ReentrantLock lock;
    /** 写入信号 **/
    private final Condition emptyLock;
    /** 读取信号 **/
    private final Condition fullLock;
    /** 写数据的下标 **/
    private int putIndex;
    /** 读取数据的下标 **/
    private int getIndex;
    public ArrayQueue(int size,Boolean fair){
        items = new Object[size];
        lock=new ReentrantLock(fair);
        emptyLock=lock.newCondition();
        fullLock=lock.newCondition();
    }

    public void put(T data) throws InterruptedException {
        final ReentrantLock lock = this.lock;
        /** 获取锁 写写互斥**/
        lock.lockInterruptibly();
        try {
            /** 如果当前队列已经满了，阻塞写入的线程**/
            while (count == items.length) {
                fullLock.await();
            }
            final Object[] items = this.items;
            items[putIndex] = data;
            /** 写入到最末尾时，重置写入下标**/
            if (++putIndex == items.length) {
                putIndex = 0;
            }
            count++;
            /** 写入成功，代表此时items有数据，唤醒读取的线程**/
            emptyLock.signal();
        } finally {
            lock.unlock();
        }
    }

    /**
     * 从队列读取最新数据
     * @return T result
     * @throws InterruptedException
     */
    public T get() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        /** 获取锁 读读 读写互斥 **/
        lock.lockInterruptibly();
        try {
            /** 如果当前没有数据，阻塞线程 **/
            while (count == 0) {
                emptyLock.await();
            }
            final Object[] items = this.items;
            T data = (T) items[getIndex];
            items[getIndex] = null;
            /** 读到最末尾 重置获取的下标 **/
            if (++getIndex == items.length) {
                getIndex = 0;
            }
            count--;
            /** 唤醒写入的线程 **/
            fullLock.signal();
            return data;
        } finally {
            lock.unlock();
        }
    }
```