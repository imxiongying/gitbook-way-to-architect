# 双缓冲队列

## 为什么要引入双缓冲队列

对前一节中介绍的几种方法，都存在一个问题：在同一时刻，队列的入队和出队是互斥的，即某一刻有且仅有一个操作！

> java.util.concurrent.ArrayBlockingQueue：同一时刻只能读或写；
>
> java.util.concurrent.LinkedBlockingQueue：内部虽然使用了读写锁，实现了读写操作的锁分离，但不能在同一时刻进行双写（入队/出队）

假如我们使用两个队列，一个队列专门用来读，另一个队列专门用来写，当读队列空或写队列满时将两个队列互换，这样就可以减少锁竞争，提升写效率。

## 双缓冲队列的Java实现

```java
public class DoubleBufferQueue<T> extends AbstractQueue<T> implements Queue<T> {

    private Lock readLock = new ReentrantLock();
    private Lock writeLock = new ReentrantLock();

    /**
     * 读队列，出队的时候，从该队列取出元素
     */
    private LinkedList<T> readQueue = new LinkedList<T>();
    /**
     * 写队列，入队的时候，从该队列放入元素
     */
    private LinkedList<T> writeQueue = new LinkedList<T>();

    public DoubleBufferQueue() {
        super();
    }

    /**
     * 添加元素到队尾
     */
    @Override
    public boolean offer(T e) {
        writeLock.lock();
        try {
            return writeQueue.offer(e);
        } finally {
            writeLock.unlock();
        }
    }

    /**
     * 移除并返问队头元素
     *
     * 当读队列为空时，交换队列
     */
    @Override
    public T poll() {
        readLock.lock();
        try {
            if (readQueue.size() == 0) {
                swap();
            }

            return readQueue.poll();
        } finally {
            readLock.unlock();
        }
    }

    /**
     * 返回队头元素（不移除）
     */
    @Override
    public T peek() {
        readLock.lock();
        try {
            if (readQueue.size() == 0) {
                swap();
            }
            return readQueue.peek();
        } finally {
            readLock.unlock();
        }
    }

    /**
     * 增加元素到队尾
     */
    @Override
    public boolean add(T e) {
        writeLock.lock();
        try {
            return writeQueue.add(e);
        } finally {
            writeLock.unlock();
        }
    }

    /**
     * 批量增加元素到队尾
     */
    @Override
    public boolean addAll(Collection<? extends T> c) {
        writeLock.lock();
        try {
            return writeQueue.addAll(c);
        } finally {
            writeLock.unlock();
        }
    }

    @Override
    public Iterator<T> iterator() {
        throw new NotImplementedException();
    }

    @Override
    public int size() {
        readLock.lock();
        writeLock.lock();
        try {
            return readQueue.size() + writeQueue.size();
        } finally {
            try {
                writeLock.unlock();
            } finally {
                readLock.unlock();
            }
        }
    }

    /**
     * 读队列和写队列交换
     */
    private void swap() {
        writeLock.lock();
        try {
            if (writeQueue.size() > 0) {
                LinkedList<T> tmp = readQueue;
                readQueue = writeQueue;
                writeQueue = tmp;
                tmp = null;
            }
        } finally {
            writeLock.unlock();
        }
    }
}
```

 测试：

```java
public class DoubleBufferQueueTest {

    public static void main(String[] args) throws Exception {
        long[] arrays = new long[]{10000, 10000 * 10, 10000 * 100, 10000 * 1000, 10000 * 10000};
        for (long count : arrays) {
            testQueue(new DoubleBufferQueue<>(), count);
            testQueue(new ArrayBlockingQueue<>((int) count), count);
            testQueue(new LinkedBlockingQueue<>(), count);
            testQueue(new LinkedBlockingDeque<>(), count);
        }
    }

    public static void testQueue(Queue<Long> queue, long count) throws Exception {
        long start = System.currentTimeMillis();
        Thread addThread = new Thread(() -> {
            for (long i = 0; i < count; i++) {
                queue.offer(i);
            }
        });

        Thread pollThread = new Thread(() -> {
            long last = -1;
            boolean lastEle = false;
            while (!lastEle) {
                Long poll = queue.poll();
                if (poll != null) {
                    if (poll - last != 1) {
                        System.out.println("not correct:::" + poll);
                    } else {
                        if (poll == count - 1) {
                            lastEle = true;
                        }
                        last = poll;
                    }
                }
            }

        });
        addThread.start();
        pollThread.start();

        addThread.join();
        pollThread.join();
        System.out.println("count:::" + count + ":::" + queue.getClass().getSimpleName() + ":::elapsed:::" + (System.currentTimeMillis() - start));
    }
}
```

 输出结果：

```text
count:::10000:::DoubleBufferQueue:::elapsed:::62
count:::10000:::ArrayBlockingQueue:::elapsed:::8
count:::10000:::LinkedBlockingQueue:::elapsed:::5
count:::10000:::LinkedBlockingDeque:::elapsed:::8

count:::100000:::DoubleBufferQueue:::elapsed:::20
count:::100000:::ArrayBlockingQueue:::elapsed:::27
count:::100000:::LinkedBlockingQueue:::elapsed:::30
count:::100000:::LinkedBlockingDeque:::elapsed:::39

count:::1000000:::DoubleBufferQueue:::elapsed:::142
count:::1000000:::ArrayBlockingQueue:::elapsed:::211
count:::1000000:::LinkedBlockingQueue:::elapsed:::272
count:::1000000:::LinkedBlockingDeque:::elapsed:::258

count:::10000000:::DoubleBufferQueue:::elapsed:::1220
count:::10000000:::ArrayBlockingQueue:::elapsed:::2965
count:::10000000:::LinkedBlockingQueue:::elapsed:::2683
count:::10000000:::LinkedBlockingDeque:::elapsed:::3118

count:::100000000:::DoubleBufferQueue:::elapsed:::11736
count:::100000000:::ArrayBlockingQueue:::elapsed:::29265
count:::100000000:::LinkedBlockingQueue:::elapsed:::27800
count:::100000000:::LinkedBlockingDeque:::elapsed:::34998
```

## 双缓冲队列的扩展

上述实现中，相当于在队列内部使用了两个容器，并使用两个锁来进行读写（这里的读是指移除元素）分离。但是，对于写/读操作而言，同一时刻，仅仅分别支持一个线程进行操作，即该双写队列的并发度为2，假如多个线程同时进行插入（或同时删除），仍然会产生大量的锁竞争。

在Java1.7版本的ConcurrentHashMap的实现中，有一种思想是分段锁。套用到这里，我们可以实现一种类似的分段队列：在队列内部，使用多个队列，其中一部分队列用于读，另一部分队列用于写，这样就可以提高同时插入或删除的线程并发数量。

## 参考

[双缓冲队列](https://www.jianshu.com/p/3724d4c1b17b)

[brucelt1993/**companyCode**](https://github.com/brucelt1993/companyCode/blob/dbfed7aa55eb81461140d0ff5044cf38c8fdea72/test_thread/com/bruce/thread2/DoubleBufferQueue.java)**：**代码来源

