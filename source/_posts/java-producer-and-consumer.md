title: java producer and consumer
date: 2015-12-08 20:34:30
tags: [java, producer-consumer]
categories: java
---

### 生产者消费者模式（以下简称PC)

生产者消费者模式是通过一个容器来解决生产者和消费者的强耦合问题。生产者和消费者彼此之间不直接通讯，而通过阻塞队列来进行通讯，所以生产者生产完数据之后不用等待消费者处理，直接扔给阻塞队列，消费者不找生产者要数据，而是直接从阻塞队列里取，`阻塞队列就相当于一个缓冲区`，平衡了生产者和消费者的处理能力。
 
<!--more-->

#### 为什么使用PC模式
- 解耦
- 缓冲
- 兼容不同端的处理能力差异

### Java实现方式

#### 1. wait()/nofify()

简单实现：

``` java
// 简单实现
public class PCQueueUsingWait<T> {
    
    private final int MAX_CAPACITY = 5;
    private Object[] items = new Object[MAX_CAPACITY];
    
    private int putptr, takeptr;
    private int count; // count total number of items
    
    public void put(T x) throws InterruptedException {
        synchronized(items) {
            /*
             * 唤醒之后，可能还会被其他put线程抢占，从而导致full，因此需要用while判断
             */
            while(count == items.length) {
                System.out.println("queue is full, please wait for consumer to take");
                items.wait();
            }
            
            items[putptr] = x;
            if (++putptr == items.length) {
                putptr = 0;
            }
            ++count;
            items.notifyAll(); // 对比PCQueueUsingLock,这里的实现没有区分full和empty的条件，因此需要notifyAll，否则会导致put之后唤醒的依旧是producer
        }
    }
    
    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        synchronized(items) {
            /*
             * 唤醒之后，可能还会被其他take线程抢占，从而导致empty，因此需要用while判断
             */
            while(count == 0) {
                System.out.println("queue is empty, please wait for producer to put");
                items.wait();
            }
            Object x = items[takeptr];
            if (++takeptr == items.length) {
                putptr = 0;
            }
            ++count;
            items.notifyAll(); // 同理
            return (T)x;
            /*
             * 这里亮神提出一个问题：是否需要再finally中使用notifyAll？ —— NO
             * 1. synchronized会自动释放锁（包括异常情况下），除非遇到blocked。yet，blocked这种情况，finally也没有办法啊~~
             * 2. notifyAll之后只会让线程进入到获取锁的等待队列中，还需要等到syn块结束之后，其他线程才能竞争到锁，因此不会出现return之前其他线程就执行的情况
             * 3. 对比：Lock需要手动去释放，因此为了保证异常情况下也能够正常释放，需要通过finally块来unLock
             */
        }
    }
}
```

改进：(TODO==)

``` java

```


#### 2. ReentrantLock - Condition

``` java
import java.util.Random;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class PCQueueUsingLock<T> {
    
    // max capacity for queue
    private final int MAX_CAPACITY = 5;
    private final Object[] items = new Object[MAX_CAPACITY];
    
    
    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    
    
    private int putptr, takeptr;
    private int count; // count total number of items
    
    
    public void put(T x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                System.out.println("queue full, wait for consumer");
                notFull.await(); // condition not met, make the thread to await
            }
            items[putptr] = x;
            if (++putptr == items.length) {
                putptr = 0;
            }
            ++count;
            notEmpty.signal(); // 唤醒所有的线程没有意义，因为最终只有一个能够执行
            
            // test 
            System.out.print("after put: ");
            for(Object obj : items) {
                System.out.print(obj + " ");
            }
            System.out.println();
            
        } finally {
            lock.unlock(); // must in finally !!!
        }
    }
    
    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                // test
                System.out.println("queue empty, wait for producer");
                notEmpty.await();
            }
            Object x = items[takeptr];
            items[takeptr] = null; // set reference to null
            if (++takeptr == items.length) {
                takeptr = 0;
            }
            --count;
            notFull.signal();
            
            // test 
            System.out.print("after take: ");
            for(Object obj : items) {
                System.out.print(obj + " ");
            }
            System.out.println();
            
            return (T)x;
        } finally {
            lock.unlock(); // must use finally !!!
        }
    }
    
    
    public static void main(String[] args) {
        Random rand = new Random(System.currentTimeMillis());
        final int round = 10;
        
        PCQueueUsingLock<Integer> pcQueue = new PCQueueUsingLock<>();
        final CountDownLatch startGate = new CountDownLatch(1);
        
        Thread p = new Thread(){
            public void run() {
                try {
                    startGate.await();  // wait 
                    for(int i=0; i<round; i++) {
                        pcQueue.put(rand.nextInt(10));
                        //Thread.sleep((long) (3000*Math.random()));
                    }
                } catch (InterruptedException e) {
                   Thread.currentThread().interrupt(); // best practice: reset interrupt flag
                }
            }
        };
        
        Thread c = new Thread(){
            public void run() {
                try {
                    startGate.await();  // wait 
                    for(int i=0; i<round; i++) {
                        pcQueue.take();
                        Thread.sleep((long) (3000*Math.random()));
                    }
                } catch (InterruptedException e) {
                   Thread.currentThread().interrupt(); // best practice: reset interrupt flag
                }
            }
        };
        
        p.start();
        c.start();
        
        startGate.countDown();  // start together
    }
}
```
测试结果

``` java
queue empty, wait for producer
after put: 4 null null null null 
after put: 4 2 null null null 
after put: 4 2 7 null null 
after put: 4 2 7 5 null 
after put: 4 2 7 5 8 
queue full, wait for consumer
after take: null 2 7 5 8 
after put: 4 2 7 5 8 
queue full, wait for consumer
after take: 4 null 7 5 8 
after put: 4 1 7 5 8 
queue full, wait for consumer
after take: 4 1 null 5 8 
after put: 4 1 8 5 8 
queue full, wait for consumer
after take: 4 1 8 null 8 
after put: 4 1 8 8 8 
queue full, wait for consumer
after take: 4 1 8 8 null 
after put: 4 1 8 8 5 
after take: null 1 8 8 5 
after take: null null 8 8 5 
after take: null null null 8 5 
after take: null null null null 5 
after take: null null null null null 
```


#### 3. BlockingQueue 
``` java
// 简单封装即可
public class PCQueueUsingBlockingQueue<T> {
    
    private final int MAX_CAPACITY = 10;
    private LinkedBlockingQueue<T> items = new LinkedBlockingQueue<>(MAX_CAPACITY);
    
    public void put(T x) throws InterruptedException {
        items.put(x); // offer() will not block, while put() will block if queue is full.
    }
    
    public T take() throws InterruptedException {
        T x = items.take(); // poll() will not block, will take() will block if queue is empty.
        return x;
    }
}
```
LinkedBlockingQueue的put内部实现(采用了ReentrantLock的方式)

``` java
 public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();   // 空元素判断
        // Note: convention in all put/take/etc is to preset local var
        // holding count negative to indicate failure unless set.
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count; // 使用atomic的方式计数，保证并发put+take下的count统计正确
        putLock.lockInterruptibly(); // 可以被中断的lock
        try {
            /*
             * Note that count is used in wait guard even though it is
             * not protected by lock. This works because count can
             * only decrease at this point (all other puts are shut
             * out by lock), and we (or some other waiting put) are
             * signalled if it ever changes from capacity. Similarly
             * for all other uses of count in other wait guards.
             */
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }
```

LinkedBlockingQueue的offer内部实现（非阻塞，立刻返回）

``` java
 public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        final AtomicInteger count = this.count;
        if (count.get() == capacity)
            return false; // 立刻返回false
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        putLock.lock(); // 不可中断，why？
        try {
            if (count.get() < capacity) {
                enqueue(node);
                c = count.getAndIncrement();
                if (c + 1 < capacity)
                    notFull.signal();
            }
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return c >= 0;
    }
```

#### 4. Semaphore

``` java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Semaphore;

public class PCQueueUsingSemaphore {
    private List<Object> items = new ArrayList<Object>(10);
    /*
     * 1. mutex保证存取缓冲区时必须是线程互斥的
     * 2. isFull保证缓冲区最多元素为initPermits，初始值代表缓冲区开始可以存放多少元素
     * 3. isEmpty保证缓冲区为0是阻塞，初始值代表缓冲区开始有多少元素
     * 4. = 也就是isFull和isEmpty的初始化值加起来等于缓冲区的大小
     * 
     * 5. 注意不同的semaphore的顺序，否则会出现并发问题
     *      - isFull的信号量可以并发获得
     *      - 但take和put实际操作时，必须只能有一个线程，因此mutex的permit=1
     * 
     * 6. 使用semaphore的好处：
     *      - 避免采用wait\notify等底层机制，封装更完善
     *      - 可以避免手动判断缓冲区的当前大小是否满或空，（Condition需要）
     *      - 借助了AQS，似乎效率上得到了优化？？？？
     */
    Semaphore mutex = new Semaphore(1); // mutex put or take
    Semaphore isFull = new Semaphore(10); // 缓冲区最多允许10个
    Semaphore isEmpty = new Semaphore(0); // 缓冲区初始值为0
    
    public void put(Object x) throws InterruptedException {
        isFull.acquire(); // 大于0,意味着还有permit可以使用，缓冲区未满
        try {
            mutex.acquire(); // acquire = ++
            items.add(x);  // release = --
        } finally {
            mutex.release();
            isEmpty.release();
        }
    }
    
    public Object take() throws InterruptedException { 
        Object x = null;
        isEmpty.acquire();
        try {
            mutex.acquire();
            /* 注意List的remove定义：
             * Removes the element at the specified position in this list (optional operation). 
             * Shifts any subsequent elements to the left (subtracts one from their indices). 
             * Returns the element that was removed from the list.
             */
            x = items.remove(0);
            return x;
        } finally {
            mutex.release();
            isFull.release();
        }
    }
}
```

#### 5. LockSupport

LockSupport可以通过park(thread)和unpark(thread)，精确地指定阻塞和唤醒线程。但是貌似就欠缺了wait/notify能够让线程在一个object上等待的接口，因此我考虑要实现PC，需要自己维护一个thread的队列才可以。

#### 6. PipedInputStream / PipedOutputStream


### 应用场景

- 需要处理任务时间比较长的场景：
    + 附件上传
    + 远程接口查询数据  
    + Java线程池

### 线程池中如何实现PC模式

### 更高效的考虑

1. putLock与takeLock分离（jdk LinkedBlockingQueue中的实现方式）
2. 如果能够直接处理，则直接被consumer取走，不需要再存储到queue中，减少复制的开销

### 队列的循环数组方式实现


### 参考
[1] [聊聊并发-生产者消费者模式](http://www.infoq.com/cn/articles/producers-and-consumers-mode)

[2] [生产者消费者问题的实现方式](http://java--hhf.iteye.com/blog/2064926)




