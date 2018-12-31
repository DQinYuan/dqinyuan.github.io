---
layout: post
title: 图解java.util.concurrent源码 （七）ArrayBlockingQueue
date: 2018-12-30
categories: java
tags: java 并发
cover: https://upload-images.jianshu.io/upload_images/10192684-8e7db4eba0be45d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
---

# 引言
---

在并发编程中经常需要进行生产者消费者之间的同步，此时我们最经常使用的同步工具就是有界阻塞队列（BlockingQueue）了，这篇文章和下一篇文章将分别分析最经常使用的两个有界队列，ArrayBlockingQueue和LinkedBlockingQueue的原理，然后对比他们的性能以及使用场景。

# BlockingQueue接口
---

BlockingQueue接口定义了juc中阻塞队列的标准，juc中的阻塞队列一般提供下列四种入队方法：

| 方法签名        | 特点    |
| --------   | :-----   | 
| `boolean add(E e)`        | 如果队列中还有剩余空间，则入队后返回true，否则立即抛出`IllegalStateException`异常，可以看出这个方法的返回值只有可能是`true`。如果是有界的阻塞队列，官方文档强烈建议使用`offer`方法代替`add`方法      | 
| `boolean offer(E e)`        | 如果队列还有剩余空间，则入队后返回true，否则返回false      |  
| `boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException`        | 同上，只不过会等待指定的时间，如果在指定的时间内队列中仍然没有剩余的空间可以入队，则返回false，否则返回true      | 
| `void put(E e) throws InterruptedException`        | 调用该方法的线程将一直阻塞直至入队成功      |

 下列三种出队方法：

| 方法签名        | 特点    |
| --------   | :-----   | 
| `E poll();`        |如果队列非空则立即将队头元素出队返回，否则返回null     | 
| `E poll(long timeout, TimeUnit unit) throws InterruptedException;`        |同上，只不过会等待指定的时间，如果指定的时间内队列仍然为空，则返回null，否则将队头元素出队返回      |  
| `E take() throws InterruptedException`        | 调用该方法的线程将一直阻塞直到队列非空，然后将队头出队返回    |

总结个比较好记：BlockingQueue提供了put和take一对阻塞方法，以及offer和poll一对非阻塞方法。

注：因为add方法文档不推荐，所以就忽略了。还有offer和poll并不是真正的非阻塞，下面从源码中我们将会看到，其实它们是基于ReentrantLock进行同步的，在竞争激烈时依旧有可能因为争抢不到锁而阻塞住。

# 数据结构基础：循环队列
---

循环队列其实就是用数组实现一个队列的结构，为了用数组模拟出队列的行为，还需要额外维护下面三个变量：

 - takeIndex: 队头在数组中的下标
 - putIndex: 队尾元素的下一个位置，即下一个元素入队的位置
 - count: 计数器，统计队列中元素数目

之所以叫循环队列其实就是因为它逻辑上把数组首尾相接成为一个环形：

![循环队列](https://upload-images.jianshu.io/upload_images/10192684-8e7db4eba0be45d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 - 入队：先确认队列未满，然后将元素放入数组的putIndex下标处，将putIndex加1，再将count加1
 - 出队：先确认队列非空，然后将数组在takeIndex处的元素返回，然后将takeIndex加1，再将count减1

需要注意的是putIndex和takeIndex不能加得超过了数组的边界，我们这里不采用取模的方法，而是在其一到达数组边界处就将其置为0，方法如下：

```java
    /**
     * Circularly increment i.
     * 每当队列满时将其置0
     */
    final int inc(int i) {
        return (++i == items.length) ? 0 : i;
    }
```

有了上面这些知识后，我们就可以尝试自己实现一个循环队列了：

```java
public class SimpleRoundQueue<T> {

    private Object[] items;

    private int putIndex;
    private int takeIndex;
    private int count;

    public SimpleRoundQueue(int size){
        items = new Object[size];
    }

    private int inc(int i){
        return (++i == items.length)? 0: i;
    }

    public boolean enQueue(T item){
        if ( count == items.length ){
            return false;
        }

        items[putIndex] = item;
        putIndex = inc(putIndex);
        count++;

        return true;
    }

    public T deQueue(){
        if ( count == 0 ){
            return null;
        }

        count--;

        T item = (T) items[takeIndex];

        takeIndex = inc(takeIndex);
        return item;
    }

    public int length(){
        return count;
    }

    //测试队列实现的正确性
    public static void main(String[] args) {
        SimpleRoundQueue<Integer> roundQueue = new SimpleRoundQueue<Integer>(3);

        roundQueue.enQueue(4);
        roundQueue.enQueue(19);
        System.out.println("length:" + roundQueue.length()); //2
        System.out.println(roundQueue.deQueue()); //4

        roundQueue.enQueue(199);
        System.out.println(roundQueue.deQueue());//19

        System.out.println("length:" + roundQueue.length()); //1

        roundQueue.enQueue(19990);
        roundQueue.enQueue(1888);

        System.out.println("length:" + roundQueue.length()); //3

        System.out.println(roundQueue.enQueue(10));  //false

        System.out.println(roundQueue.deQueue());//199
        System.out.println(roundQueue.deQueue());//19990
        System.out.println(roundQueue.deQueue());//1888
        System.out.println(roundQueue.deQueue());//null
    }

}
```

看起来很简单的数据结构，但是有的时候面试官让你手写，一紧张就写不出来了，还是记住为好。

# ArrayBlockingQueue实现分析
---

其实你看完上面我实现的那个循环队列，ArrayBlockingQueue的源码就已经看完了一大半了， 其实它就是在循环队列的出队和入队操作上都加了一把大锁而已，下图是一个简单粗暴的伪代码：

![ArrayBlockingQueue源码大体结构](https://upload-images.jianshu.io/upload_images/10192684-4409527caf0db086.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 构造方法

先看看构造方法干了什么：

```java
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        /**
         * 新建了两个Condition对象，分别是空条件和满条件
         */
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
```

先新建了作为容器的对象数组，然后新建了之前图中所示的那把大锁，在从锁中建立了生产者消费者问题中很常见的两个条件：非空条件（notEmpty）和未满条件（notFull）。

### 入队实现

然后看offer操作：

```java
    public boolean offer(E e) {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count == items.length) //队列已满
                return false;
            else { //队列尚未满
                insert(e);//循环队列入队
                return true;
            }
        } finally {
            lock.unlock();
        }
    }
```

其中insert方法的代码其实就是循环队列的入队操作：

```java
    private void insert(E x) {
        items[putIndex] = x;
        putIndex = inc(putIndex);
        ++count;
        //唤醒一个等待非空条件的线程
        notEmpty.signal();
    }
```

可以看出相比我们之前的实现的循环队列的入队方法就多了一句`notEmpty.signal()`，会唤醒一个等待在“非空”条件上的线程。

`put`方法的实现和`offer`几乎是一样的，只是发现队列满了之后会`await`在未满这个条件上：

```java
    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)  //等待直到队列中有剩余空间
                notFull.await();
            insert(e);
        } finally {
            lock.unlock();
        }
    }
```

`add`方法就不说了，因为它其实调用的就是offer方法，offer方法返回false就抛异常。

再提一下带有时间限制的`offer`方法的实现：

```java
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        checkNotNull(e);
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            //ReentrantLock文档中推荐的awaitNanos的标准使用方法
            while (count == items.length) {
                if (nanos <= 0)
                    return false;
                nanos = notFull.awaitNanos(nanos);
            }
            insert(e);
            return true;
        } finally {
            lock.unlock();
        }
    }
```

其实和不带时间限制的offer方法大同小异，只不过这里阻塞用的方法是`awaitNanos`，这里之所以要使用一个while循环直到`nanos`减至0，是因为`awaitNanos`存在“伪唤醒”的问题，可能会在规定的等待时间还没到的时候（for no reason）就返回，返回值是剩余的等待时间，如果你一定想要线程等待这么久的时间的话，`ReentrantLock`文档中推荐的写法是：

```java
            while (count == items.length) {
                if (nanos <= 0)
                    return false;
                nanos = condition.awaitNanos(nanos);
            }
```

在我之前一篇分享`CyclicBarrier`源码的[文章](https://www.dqyuan.top/2018/12/25/cyclicbarrier.html)中也提到了这个"伪唤醒"问题。

### 出队实现

`poll`方法：

```java
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //count == 0判断队列是否为空
            return (count == 0) ? null : extract();
        } finally {
            lock.unlock();
        }
    }
```

`extract`方法其实就是循环队列的出队操作:

```java
    private E extract() {
        final Object[] items = this.items;
        E x = this.<E>cast(items[takeIndex]);
        items[takeIndex] = null;
        takeIndex = inc(takeIndex);
        --count;
        /**
         * 唤醒一个等待剩余空间的线程
         */
        notFull.signal();
        return x;
    }
```

相比循环队列的出队操作，只多了一行`notFull.signal();`。

`take`方法的实现大同小异：

```java
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                //如果队列为空,则阻塞在"非空"条件上
                notEmpty.await();
            return extract();
        } finally {
            lock.unlock();
        }
    }
```

# End
---

`ArrayBlockingQueue`的实现非常简单，但是却涉及到“循环队列”以及“生产者消费者”问题两个基础知识点，有的时候面试官会让你手写，所以个类的代码最好能够记住，之前我只顾着研究juc中比较复杂的类了，结果面试官让写这个简单的类，反而没写出来，很尴尬。

另外，`ArrayBlockingQueue`也是学习`ReentrantLock`使用方法的好材料。






