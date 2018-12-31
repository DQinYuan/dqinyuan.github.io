---
layout: post
title: 图解java.util.concurrent源码 （八）LinkedBlockingQueue
date: 2018-12-31
categories: java
tags: java 并发
cover: https://upload-images.jianshu.io/upload_images/10192684-42479ad651d427f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
---

# 引言
---

[上一篇文章](https://www.dqyuan.top/2018/12/30/arrayblockingqueue.html)中分析了ArrayBlockingQueue的源码，说好这一篇文章中要继续分析LinkedBlockingQueue的源码并且对比他们的使用场景，在看这篇文章之前建议先看一下上一篇文章。

# LinkedBlockingQueue数据结构
---

LinkedBlockingQueue底层是一个链表结构，入队时直接将节点连接在链表的后面，出队时直接将头结点剔除即可，核心的变量如下：

 - capacity: 容量，队列的最大大小
 - count: 计数器，用于计算当前队列大小
 - head: 头结点
 - last: 尾节点

初始化时：
    ![初始化时](https://upload-images.jianshu.io/upload_images/10192684-bf4c155507f1b6e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

插入obj1时：

![obj1插入时](https://upload-images.jianshu.io/upload_images/10192684-547eeb81e2861865.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里的链表尾插的代码写得非常简洁，情不自禁提前拿出来展示一下：

```java
    /**
     * Links node at end of queue.
     *
     * @param node the node
     */
    private void enqueue(Node<E> node) {
        // assert putLock.isHeldByCurrentThread();
        // assert last.next == null;
        last = last.next = node;
    }
```

相当于先把要插入的节点赋值给last节点的next字段（`last.next = node`），然后因为这个赋值表达式也是有值的（就是`node`），然后将它直接赋给了last作为尾节点。如果面试时能写出这么漂亮的尾插会不会有加分呢？

插入obj2时：

![插入obj2时](https://upload-images.jianshu.io/upload_images/10192684-113d022942e045f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

进行一次出队后：

![经过一次出队后](https://upload-images.jianshu.io/upload_images/10192684-39bb77702e376b7f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

头结点里面的元素始终是null。

可以看出，LinkedBlockingQueue就是一个标准的链表队列的实现，而ArrayBlockingQueue也是一个标准的基于数组的队列实现，实现非常简洁，是拿来学习数据结构的好材料，后悔以前学习数据结构的时候怎么没有发现。

# LinkedBlockingQueue实现分析
---

### 构造方法

```java
    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }
```

确定容量，并且将头尾指针都指向同一个内容为null的节点，就是我上一节中“初始化时”那张图画得那样。

LinkedBlockingQueue也可以在创建时不指定容量（ArrayBlockingQueue则必须指定容量）：

```java
    /**
     *
     * 默认的队列容量为int的最大值
     * Creates a {@code LinkedBlockingQueue} with a capacity of
     * {@link Integer#MAX_VALUE}.
     */
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }
```

可以看出默认的容量为int的最大值，相当于无限容量了。


### 锁

LinkedBlockingQueue相比ArrayBlockingQueue最大的特色在与它有两把锁，一把用来锁入队，一把用于锁出队：

```java
    /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();
```

"非空"条件（`notEmpty`）是从“出队锁”中产生的，而"未满"条件（`notFull`）是从“入队锁”中产生的，与ArrayBlockingQueue中所有Condition都产生自同一把锁是不同的。

### 入队操作

入队的几个方法的实现都是大同小异的，我以offer方法为例（图片可能有点小，[点击查看原图](https://upload-images.jianshu.io/upload_images/10192684-d3a27309ce18dfa5.png)）：

![入队流程](https://upload-images.jianshu.io/upload_images/10192684-d3a27309ce18dfa5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

流程图中的步骤我都在下面的代码注释中标注出（一对应①，二对应②，以此类推）：

```java
    public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        final AtomicInteger count = this.count;
        /**
         * 一:是出于性能考虑,先进行一次无锁的判断
         */
        if (count.get() == capacity)
            return false;
        int c = -1;
        Node<E> node = new Node(e);
        final ReentrantLock putLock = this.putLock;
        /**
         * 二：加锁
         */
        putLock.lock();
        try {
            /**
             * 三：查看加锁后是否还有剩余空间
             */
            if (count.get() < capacity) {
                /**
                 * 四:如果加锁后还有剩余空间,则将新建的节点插入尾部
                 */
                enqueue(node);
                c = count.getAndIncrement();
                /**
                 * 五:如果发现还有剩余空间则再唤醒一个入队线程,相比ArrayBlockingQueue需要加这一步的原因是锁的粒度比较细
                 *
                 * 有可能在出队还未完成时就有数个元素入队,此时就必须要靠入队线程来传播
                 */
                if (c + 1 < capacity)//发现有剩余空间
                    //唤醒一个等待剩余空间的线程
                    notFull.signal();
            }
        } finally {
            putLock.unlock();
        }
        /**
         * 六:c为0说明这是在队列空以后放入的第一个元素,则唤醒一个等待非空条件的线程
         */
        if (c == 0)
            signalNotEmpty();

        /**
         * c大于等于0才说明中间没有发生异常
         */
        return c >= 0;
    }
```


下面提一下代码中一些值得注意的地方（标号对应流程图中的标号）。

#### ① 队列已满（无锁）

代码在加锁之前会先进行一次无锁的队列容量判断，其实没有这一段代码程序也是可以正确运行的，我们认为这里主要是为了性能优化的考虑，在队列拥挤时能够避免线程加解锁的次数。

`put`方法则没有这一段代码。

#### ④ 将节点插入尾部

这里的`enqueue`方法就是上一节展示的链表尾插代码。

#### ⑤ 还有剩余空间的话则再唤醒一个入队线程

在`ArrayBlockingQueue`的源码中，"入队线程"只需要唤醒"出队线程"就可以了，这里为什么"入队线程"还要唤醒"入队线程"呢？

原因在于这里的锁的粒度比`ArrayBlockingQueue`要细，`ArrayBlockingQueue`出队和入队都是同一把锁，在出队的时候，入队线程就都阻塞住了，不可能出现同时出入的情况，所以只需要在每出队一个元素时就只唤醒一个入队线程，这种严格的同步关系能够得到保证。

在这里锁的粒度变细了，出队锁和入队锁分成了两把，出入可以同时进行，上面说的那种严格的同步关系就无法保证了，所以这里要增加一个对己方（出队线程）的通知（`notFull.signal()`）。待会看出队操作源码时，会看到类似的对己方的通知。

#### ⑥ 如果发现在自己入队之前队列是空的，则唤醒一个出队线程

这里主要是调用了`signalNotEmpty`方法，如下：

```java
    private void signalNotEmpty() {
        /**
         * 这里获得锁仅仅是为了唤醒线程
         */
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }
```

因为和出队不是同一把锁，所以这里必须要先获取对面（出队线程）的锁再唤醒一个出队线程。

其他的入队方法都是类似的一套模板。

### 出队操作

出队其实和入队的代码完全是对称的：

```java
    public E poll() {
        final AtomicInteger count = this.count;
        /**
         * 一：出于性能考虑,先进行一次无锁的非空判断
         */
        if (count.get() == 0)
            return null;
        E x = null;
        int c = -1;
        final ReentrantLock takeLock = this.takeLock;
        /**
         * 二：加锁
         */
        takeLock.lock();
        try {
            /**
             * 三：查看加锁后队列是否非空
             */
            if (count.get() > 0) {
                /**
                 * 四：如果加锁后队列仍然非空,则头节点出队
                 */
                x = dequeue();
                c = count.getAndDecrement();
                /**
                 * 五：如果发现还有剩余的元素,则再唤醒一个出队线程
                 */
                if (c > 1)
                    notEmpty.signal();
            }
        } finally {
            takeLock.unlock();
        }

        /**
         * 六：如果原先队列是满的,则唤醒一个等待剩余空间的线程
         */
        if (c == capacity)
            signalNotFull();
        return x;
    }
```

三中的出队方法`dequeue`如下：

```java
    private E dequeue() {
        // assert takeLock.isHeldByCurrentThread();
        // assert head.item == null;
        Node<E> h = head;
        Node<E> first = h.next;
        h.next = h; // help GC
        head = first;
        E x = first.item;
        first.item = null;
        return x;
    }
```

就是将head指针移动到了下一个Node节点并且将其中的item置位了null。



# 对比ArrayBlockingQueue
---

LinkedBlockingQueue和ArrayBlockingQueue的差异如下：

 - 一个底层是链表，一个底层是数组
 - LinkedBlockingQueue中出队和入队各自有一个锁，出入队操作互不冲突，可以同时进行，而ArrayBlockingQueue出队和入队都是一把锁，不能同时出入。

从他们的实现上猜测他们的优劣如下：

| 类        | 优势    |  劣势  |
| --------   | :-----   | :---- |
| LinkedBlockingQueue        | 锁的粒度更细，高并发下应该会有更好的吞吐量      |   在频繁出入队的情况下，需要频繁地创建和删除Node节点，内部的实现代码上也比ArrayBlockingQueue要复杂一些    |
| ArrayBlockingQueue        |   不需要额外封装node对象，内部实现代码简单  |   锁的粒度粗，高并发下影响吞吐量    |


# 性能测试
---

性能测试代码如下：

```java
import java.util.concurrent.*;

/**
 * 结论：在小数据量大竞争的情况下使用LinkedBlockingQueue更快
 *       但是在大数据量的情况下，ArrayBlockingQueue更快
 */
public class BlockingQueueBenchmark {

    /**
     * 每个线程读写的数据数量
     */
    private static final int COUNT_PER_THREAD = 10000;

    private static class Test {
        private int data;

        public Test(int data) {
            this.data = data;
        }
    }


    private static void execute(BlockingQueue<Test> queue, ExecutorService pool, CountDownLatch latch,
                                int poolSize, boolean write) {
        for (int i = 0; i < poolSize; i++) {
            pool.execute(
                    () -> {
                        for (int j = 0; j < COUNT_PER_THREAD; j++) {
                            try {
                                if (write) {
                                    queue.put(new Test(j));
                                } else {
                                    queue.take();
                                }
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                        }

                        latch.countDown();
                    }
            );
        }
    }


    public static void main(String[] args) throws InterruptedException {
        int cpuNum = Runtime.getRuntime().availableProcessors();

        /**
         * 使用CPU一半的核进行读,另一半的核进行写
         */
        ExecutorService writePool = Executors.newFixedThreadPool(cpuNum / 2);
        ExecutorService readPool = Executors.newFixedThreadPool(cpuNum / 2);

        BlockingQueue<Test> arrayQueue = new ArrayBlockingQueue<>(1000);

        CountDownLatch latch = new CountDownLatch(cpuNum);

        long start = System.currentTimeMillis();

        execute(arrayQueue, writePool, latch, cpuNum / 2, true);
        execute(arrayQueue, readPool, latch,cpuNum / 2, false);

        latch.await();

        System.out.println("arrayQueue time used:" + (System.currentTimeMillis() - start) + "ms");

        latch = new CountDownLatch(cpuNum);
        start = System.currentTimeMillis();
        BlockingQueue<Test> linkedQueue = new LinkedBlockingQueue<>(cpuNum);

        execute(linkedQueue, writePool, latch, cpuNum / 2, true);
        execute(linkedQueue, readPool, latch, cpuNum / 2, false);

        latch.await();

        System.out.println("linkedQueue time used:" + (System.currentTimeMillis() - start) + "ms");
    }


}
```

因为我的电脑是12核的，所以对于我的电脑来说`cpuNum / 2 == 6`，采用6个线程入队，6个线程出队。

通过对每个线程出入队数目参数（`COUNT_PER_THREAD `）的控制，得到以下统计数据(总计耗时)：


| 每个线程出入数目\类        | ArrayBlockingQueue    |  LinkedBlockingQueue  |
| --------   | :-----   | :---- |
| 100        |  38ms     | 2ms     |
| 1000       |  43ms  |  10ms    |
| 10000       |  57ms  |  57ms    |
| 100000       |  107ms  | 339ms     |
| 1000000       |465ms    |  2874ms    |

![ArrayBlockingQueue和LinkedBlockingQueue性能测评](https://upload-images.jianshu.io/upload_images/10192684-42479ad651d427f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


可以看出在数据量小的情况下，`LinkedBlockingQueue`还是很优势的，在每个线程出入100个元素，LinkedBlockingQueue要快接近20倍，但是随着数据量的增长，`LinkedBlockingQueue`每次都要新建Node节点，并且两把锁操作复杂的劣势逐渐显现出来，在每个线程出入1000000个元素时，相比`ArrayBlockingQueue`要慢了六倍多。

**结论：在小数据量大竞争的情况下使用LinkedBlockingQueue更快，但是在大数据量的情况下，ArrayBlockingQueue更快**

# End
---

个人觉得以生产者消费者问题为例，生产者产量不是很大的情况下用LinkedBlockingQueue更加合适，而在产量巨大（万级别以上）的情况下，用ArrayBlockingQueue更加合适。

另外我比较疑惑的一点是为什么不能实现一个出入队锁分离的`ArrayBlockingQueue`呢？这样就可以兼具两者的优势了，这个我还没有想明白。