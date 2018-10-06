---
layout: post
title: 图解java.util.concurrent源码（四） 可重入读写锁（ReentrantReadWriteLock）
date: 2018-10-07
categories: java
tags: java 并发
cover: https://upload-images.jianshu.io/upload_images/10192684-2036bff482685f3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
---

# 引言
---
上一篇文章所讲述的ReentrantLock和Semophore分别是AQS在独占模式和共享模式的经典实现。而这次要分享的ReentrantReadWriteLock则是混合了独占共享模式的经典实现。

在读这篇文章之前，你最好已经理解了AQS和ReentrantLock，如果你还不理解的话，可以分别见本系列的[第一篇文章](https://www.dqyuan.top/2018/09/06/abstractqueuesynchronizer.html)和[第三篇文章](https://www.dqyuan.top/2018/09/24/reentrantlock-semaphore.html)

# 读锁和写锁
---
从一个ReentrantReadWriteLock对象中可以获得两把锁，分别是读锁和写锁，如下：
```java
    private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    private Lock readLock = lock.readLock();

    private Lock writeLock = lock.writeLock();
```

它们的互斥关系如下：

|         | 读锁    |  写锁  |
| --------   | :-----:   | :----: |
| **读锁**        | 兼容      |   互斥    |
| **写锁**        | 互斥      |   互斥    |

只有读锁与读锁之间是兼容的，也就是说可以多个线程同时持有该类型的锁。

# 锁降级（Lock Downgrading）
---

ReentrantReadWriteLock的一个重要特性是锁降级。

锁降级的含义如下：

![锁降级](https://upload-images.jianshu.io/upload_images/10192684-2036bff482685f3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

按照如上的顺序编写代码即可将线程持有的写锁降级为读锁。

典型的锁降级的代码如下：

```java
        ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

        Lock readLock = lock.readLock();

        Lock writeLock = lock.writeLock();
        //获取写锁
        writeLock.lock();
        try {
            //进行一些写任务
            //...

            //获取读锁
            readLock.lock();
        } finally {
            //释放写锁
            writeLock.unlock();
        }

        try {
            //进行一些读任务
            //...
        } finally {
            //释放读锁
            readLock.unlock();
        }
```

锁升级是不被允许的，可以把上面的代码的读写锁顺序换一下放到IDE里运行看看效果：

```java
        ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

        Lock readLock = lock.readLock();

        Lock writeLock = lock.writeLock();
        //获取读锁
        readLock.lock();
        try {
            //...

            //获取写锁
            writeLock.lock();
        } finally {
            //释放读锁
            readLock.unlock();
        }

        try {
            //...
        } finally {
            //释放写锁
            writeLock.unlock();
        }
```

在我电脑上看的现象是，整个程序卡住不动了（其实是发生了死锁）。此时程序也不会报错，所以我们在使用ReentrantReadWriteLock时要格外小心，不能进行锁升级的操作。

## 应用场景

假设有一段必须要连续进行的任务，任务的前一半是写而后一半是读，为了提升程序的吞吐，我可以在写任务结束后将锁降级为读锁，这样其他线程就可以进入临界区进行读操作，如下图：

![锁降级写读切换](https://upload-images.jianshu.io/upload_images/10192684-26e62c59fdac3d42.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这里之所以不能使用传统的`获取写锁 -> 释放写锁 -> 获取读锁
 -> 释放读锁`的模式，是因为在`释放写锁 -> 获取读锁`的过程中可能会有其他线程抢占到写锁，修改数据，导致当前任务的写读被打断，如下图：

![传统写读切换](https://upload-images.jianshu.io/upload_images/10192684-5568dc45a8bf3f0b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 抑制写线程饥饿
---

读写锁比较容易出现的一个问题是写线程饥饿。写线程饥饿是指，假设读线程源源不断地到来，我们知道读锁与写锁是相互排斥的，那么写线程就会一直无法获取到写锁。

ReentrantReadWriteLock天然具有对这个问题的抵抗力，可以写一段代码测试一下：

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class WriteStarvation {

    private static ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    private static Lock readLock = lock.readLock();

    private static Lock writeLock = lock.writeLock();


    static class WriteThread extends Thread{
        @Override
        public void run() {
            writeLock.lock();

            try {
                System.out.println("I am writing.");
            } finally {
                writeLock.unlock();
            }
        }
    }

    static class ReadThread extends Thread{

        private int i;

        public ReadThread(int i) {
            this.i = i;
        }

        @Override
        public void run() {
            readLock.lock();
            try {
                System.out.println("I am reading " + i);
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                readLock.unlock();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new ReadThread(0).start();

        //为了确保读线程先启动
        Thread.sleep(1000);

        new WriteThread().start();

        //启动100个读线程
        for ( int i = 1; i <= 100; i++ ){
            new ReadThread(i).start();
        }
    }

}
```

我先启动了一个需要占用读锁5秒的读线程，之后主线程休息1秒后启动写线程，此时写线程肯定会因为抢不到锁的阻塞，之后还会有100个读线程（每个都需要占用5秒的读锁）源源不断地到来抢占读锁。

运行结果如下（这里只给出前五行）：

```java
I am reading 0
I am reading 1
I am writing.
I am reading 2
I am reading 3
...
```

可见读线程还是很快就得到执行了，并没有因为源源不断到来的读线程而“饥饿”。你也可以用公平的读写锁（`new ReentrantReadWriteLock(true)`）试试，看看结果有什么不同。

这个现象的产生，其实是利用了AQS中的CLH队列，让申请读锁的线程谦让申请写锁的线程，具体的实现请看下文的源码分析。


# 与AQS的联系
---
仔细看一下ReentrantReadWriteLock的获取两把锁的方法以及构造方法：
```java
    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * default (nonfair) ordering properties.
     *
     * 创建一个非公平的可重入读写锁
     */
    public ReentrantReadWriteLock() {
        this(false);
    }

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * the given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }

    public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
    public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
```

会发现ReentrantReadWriteLock有两个内部类，分别叫ReadLock和WriteLock，分别对应这个读锁和写锁。

ReadLock和WriteLock的内部都分别有一个sync对象，很显然它是AQS的一个实现，这两个锁的各种操作都是委托给sync进行的。那么这个sync来自哪里，看一下ReadLock和WriteLock的构造方法就知道，都是来自于外部类对象的成员变量sync：

```java
        protected ReadLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
```

```java
        protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
```

而ReentrantReadWriteLock的成员变量sync的值则是由构造方法中的`sync = fair ? new FairSync() : new NonfairSync();`这一句确定的。也就是说读锁和写锁用的其实都是同一个sync对象，其中写锁是以独占模式使用而读锁是以共享模式使用（具体下文再分析），如图：

![读写锁与sync.png](https://upload-images.jianshu.io/upload_images/10192684-38671c440116e7c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同时一个sync中同时实现了独占模式与共享模式的功能，所以才说ReentrantReadWriteLock实现的是混合模式的AQS。

# 同步状态
---
AQS靠一个同步状态state来协调线程，这里state需要同时记录写锁相关的信息以及读锁相关的信息，所以将其拆成两部分，高16位表示读，低16位表示写，具体含义如下：

 - 高16位：当前各个线程持有读锁的数量总和（包括重入的数量）
 - 低16位：重入写锁的次数

Sync内部类中与state的相关的方法与变量如下：

```java
        //读状态从state的第16位开始
        static final int SHARED_SHIFT   = 16;
        /**
         * 读状态的"1"，主要是为方便计算
         * 当需要给读状态加1时，直接在state上加上这个数值就是可以了
        **/
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        //state最大所能承载读状态或者写状态的数值
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
       //写状态掩码，将state与它进行与运算即可获得
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        /** 输入c都是state，左移16位即是读状态  */
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        /** 输入c都是state，与EXCLUSIVE_MASK进行与运算即可得到写状态   */
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

# 写锁
---

看了上面写状态的含义就可以猜到，写锁的实现和ReentrantLock基本上是一样的。

先看看写锁的lock与unlock方法：

```java
        public void lock() {
            sync.acquire(1);
        }
```

```java
        public void unlock() {
            sync.release(1);
        }
```

acquire和release分别是AQS在独占模式下的锁获取与释放的方法，子类需要实现相应的tryAcquire与tryRelease方法。


Sync类的tryAcquire方法：

```java
        protected final boolean tryAcquire(int acquires) {
            Thread current = Thread.currentThread();
            int c = getState();
            //w表示写锁个数
            int w = exclusiveCount(c);
            /**c!=0说明此时有其他线程持有锁（读锁或者写锁）
              *此时要么获取锁失败，要么写锁重入
             **/
            if (c != 0) {
                //c != 0 && w == 0表示存在读锁
                //current != getExclusiveOwnerThread()说明有其他线程持有写锁
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;

                 //当运行到这里时，说明是线程重入

                //不能超出写状态所能记录的最大数值
                //这里MAX_COUNT即是线程写锁可重入的最大数值
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                //直接加到低16位的写状态上面
                setState(c + acquires);
                return true;
            }
            //writerShouldBlock方法用于控制公平性,在FairSync与NonFair中有不同的实现
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;

            //运行到这里说明是该线程第一次获取写锁
            setExclusiveOwnerThread(current);
            return true;
        }
```

因为写状态位于state的低16位，所以更新state值的时候直接将state加上要更新的数值即可（`setState(c + acquires)`）

用于控制公平性的方法是writerShouldBlock方法，它是Sync类的一个抽象方法：

```java
        /**
         * 如果当前线程没有资格获得写锁，返回true，否则返回false
         */
        abstract boolean writerShouldBlock();
```

FairSync的实现如下：

```java
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
```

hasQueuedPredecessors是从AQS中继承来的一个方法，如果当前线程是等待队列的队头则返回false（即没有队列前继），否则返回true。

NonFairSync的实现如下：

```java
        final boolean writerShouldBlock() {
            return false; 
        }
```

不需要任何额外的控制，直接返回false，此时writerShouldBlock方法就形同虚设了。

Sync类的tryRelease方法如下：

```java
        protected final boolean tryRelease(int releases) {
            //当前线程必须持有写锁，否则抛出异常
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            //减少写状态的数值
            //因为写状态在低16位，直接减去即可
            int nextc = getState() - releases;
            //free为true表示当前线程的写锁已经被完全释放掉了
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }
```

非常简单，就是将state减去releases，如果此时线程已经释放了所有重入的写锁（`exclusiveCount(nextc) == 0`），则将exclusiveOwnerThread设置为null。

写锁是支持使用ConditionObject来进行线程同步的，其实现也非常常规。

WriteLock类的newCondition方法：

```java
        public Condition newCondition() {
            return sync.newCondition();
        }
```

Sync中的相关方法：

```java
        final ConditionObject newCondition() {
            return new ConditionObject();
        }
      
         /**
           *如果想要支持ConditionObject，子类所必须实现的方法
           *用于判断锁是否被当前线程独占
          **/
         protected final boolean isHeldExclusively() {
            return getExclusiveOwnerThread() == Thread.currentThread();
        }
```

比较有趣的是在ReentrantLock提供的两把锁中只有写锁（WriteLock）提供Condition对象进行线程同步，而读锁（ReadLock）是不提供的。

从上面的分析可以看出，写锁虽然代码与ReentrantLock不太一样，但是思路如出一辙，关于ReentrantLock可以见我该系列的[上一篇文章](https://www.dqyuan.top/2018/09/24/reentrantlock-semaphore.html)。

下面来学习一下稍微复杂一些的读锁。

# 读锁
---

读锁实现的总体思路是这样的：

 - 使用state的高16位表示读锁被持有的总的数量
 - 使用ThreadLocal维护每个线程重入读锁的次数

第一点很容易理解，下面来看看第二点。

Sync类中有一个叫做readHolds的成员变量，它维护着每个线程重入读锁的次数：

```java
        private transient ThreadLocalHoldCounter readHolds;
```

ThreadLocalHoldCounter是Sync的一个内部类，其实就是一个ThreadLocal：

```java
        static final class ThreadLocalHoldCounter
            extends ThreadLocal<HoldCounter> {
            public HoldCounter initialValue() {
                return new HoldCounter();
            }
        }
```

ThreadLocal中存放的是HoldCounter类型，它也是Sync的一个内部类，专门负责记录相应线程的重入次数：

```java
        static final class HoldCounter {
            //相应线程重入的次数
            int count = 0;
        
            //tid,即线程的id
            //为了避免相互引用影响垃圾回收,这里只存储了线程的tid而不存储Thread类型的引用
            final long tid = Thread.currentThread().getId();
        }
```


下面来看看实现细节。

看一下读锁的lock和unlock方法：

```java
        public void lock() {
            sync.acquireShared(1);
        }

        public  void unlock() {
            sync.releaseShared(1);
        }
```

acquireShared和releaseShared分别是AQS在共享模式下获取锁与释放锁的方法，子类需要实现tryAcquireShared和tryReleaseShared方法。

看一下Sync类的tryAcquireShared方法来理解一下上面所说的总体思路：

```java
        /**
         * 返回 >=0 表示获取锁成功
         * 返回 <0 表示失败
         * @param unused
         * @return
         */
        protected final int tryAcquireShared(int unused) {
            Thread current = Thread.currentThread();
            int c = getState();
            //一
            /**
             * 如果存在写锁且持有锁的不是当前线程，则返回-1
             */
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;

            //二
            /**
             * 读锁相关判断
             */
            int r = sharedCount(c);
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {

                 //三
                //这一段只是对于成功获得读锁的不同的处理而已，只要进入了这个判断就一定能获得读锁
                //firstReaderHoldCount表示第一个获得读锁的线程的重入次数
                if (r == 0) {   //第一个获取读锁的线程单独对待
                                //不放入readHolds以保证性能，因为这两个经常会用到做判断
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) { //第一把读锁重入
                                                     //这个的存在同样是因为区别对待第一个获得读锁的线程
                    firstReaderHoldCount++;
                } else {  //非重入读锁
                    HoldCounter rh = cachedHoldCounter;
                    //因为本代码作者认为ThreadLocal.get方法很耗时，所以缓存了一个HoldCounter
                    //firstReaderHoldCount因为经常要判断，所以单独拿出来，以免ThreadLocal.get影响效率
                    if (rh == null || rh.tid != current.getId())
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    //计数 + 1
                    rh.count++;
                }


                return 1;
            }


            //四
            return fullTryAcquireShared(current);
        }
```

fullTryAcquireShared方法：

```java
        final int fullTryAcquireShared(Thread current) {
            HoldCounter rh = null;
            //CAS循环
            for (;;) {
                int c = getState();
                //五
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)   //存在写锁且持有锁的不为当前线程则返回-1
                        return -1;
                    //没有写锁存在或者当前线程在进行锁降级
                } else if (readerShouldBlock()) {

                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        //非重入线程运行到这里肯定会抢锁失败
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != current.getId()) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                //六
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {   //首次获取读锁
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {   //重入读锁
                        firstReaderHoldCount++;
                    } else {   //第一个获得读锁的线程以外的线程获得读锁
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != current.getId())
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }
```


这两段代码的逻辑整理如下：

![tryAcquireShared整体逻辑](https://upload-images.jianshu.io/upload_images/10192684-270d784b8115656e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中终结状态（即方法返回）我都用淡红色标出了，其中的标号①②③④⑤⑥对应代码注释中的一二三四五六。

下面进行一一剖析：

## ①

对应的代码：

```java
            /**
             * 如果存在写锁且持有锁的不是当前线程，则返回-1
             */
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
```

非常简单，其中加上`getExclusiveOwnerThread() != current`判断是为了允许锁降级。

## ②

对应着判断条件：

```java
            int r = sharedCount(c);
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
```

如果能通过这个if检查，那么线程就一定能够获得锁了。

检查的后两个条件很好理解，` r < MAX_COUNT`用于防止读状态的数值超过最大值，`compareAndSetState(c, c + SHARED_UNIT)`如果该cas操作成功将state加上`SHARED_UNIT`则返回true，如果遇到竞争没能成功则返回false。

第一个条件检查`readerShouldBlock`，其作用和之前写锁中的`writerShouldBlock`一样，都是用于控制公平性的。

其中FairSync对于readerShouldBlock的实现与`writerShouldBlock`是一样的，如下：

```java
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -2274990926593161451L;
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
    }
```

而在NonFairSync中的实现则不太一样：

```java
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -8159625535654395037L;
        final boolean writerShouldBlock() {
            return false;
        }
        final boolean readerShouldBlock() {
            return apparentlyFirstQueuedIsExclusive();
        }
    }
```

apparentlyFirstQueuedIsExclusive是来源于AQS中的一个方法，源代码如下：

```java
    final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null && (s = h.next) != null && !s.isShared() && s.thread != null;
    }
```

复习一下在AQS的CLH队列中，头结点的下一个节点才是第一个等待的节点，所以上面代码中的s代表的就是第一个等待的节点，如果它不是共享模式的（即独占模式），就返回true.

所以非公平地获取读锁的线程在看到队列头是想要抢占写锁的线程时，就会将自己阻塞住（后面的那个fullTryAcquireShared方法也会做readerShouldBlock的判断，为true时返回-1）。这么做的主要原因是为了**防止写饥饿**（即申请写锁的线程长时间无法获得写锁），所以在这里读线程要进行适当的谦让。

关于读写锁在公平性与非公平性的实现上的不同，我总结了下表：

![读写锁公平性与非公平性的实现](https://upload-images.jianshu.io/upload_images/10192684-4f6265e949f276fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## ③

到达了这一步，线程就一定能够获得读锁了，而这一步所需要做的就是将state的高16位加上1（因为又有一个新的读锁被分配了出去），然后在该线程的读锁重入计数器上再加上1，最后再`return 1`，表示获得锁成功，就可以了。

听上去很简单，为什么写了这么多代码呢？：

```java
                //这一段只是对于成功获得读锁的不同的处理而已，只要进入了这个判断就一定能获得读锁
                //firstReaderHoldCount表示第一个获得读锁的线程的重入次数
                if (r == 0) {   //第一个获取读锁的线程单独对待
                                //不放入readHolds以保证性能，因为这两个经常会用到做判断
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) { //第一把读锁重入
                                                     //这个的存在同样是因为区别对待第一个获得读锁的线程
                    firstReaderHoldCount++;
                } else {  //第一个获得读锁的线程以外的线程获得读锁
                    HoldCounter rh = cachedHoldCounter;
                    //因为本代码作者认为ThreadLocal.get方法很耗时，所以缓存了一个HoldCounter
                    //firstReaderHoldCount因为经常要判断，所以单独拿出来，以免ThreadLocal.get影响效率
                    if (rh == null || rh.tid != current.getId())
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    //计数 + 1
                    rh.count++;
                }


                return 1;
```

写这么多代码，主要是为了维护在Sync类中一些成员变量，更进一步的目的是利用缓存提升性能，如下：

 - firstReader：第一个获取到读锁的线程（是指一轮中的第一个，这里的一轮指的是很多申请读锁的线程蜂拥而至，直至他们把读锁全部释放）
 - firstReaderHoldCount：第一个获取到读锁的线程重入读锁的次数
 - cachedHoldCounter：用于缓存最近一次获得读锁的线程的重入读锁次数计数器（前面所讲的HoldCounter类型），源代码中的注释是"cache for release"，可能是因为作者认为最近一次获得读锁的线程往往也是最早释放的，所以提前缓存起来，这样释放的时候就不用去ThreadLocal中找了，可以提升性能。我对这个还是比较疑惑的，因为总觉得这个假设并不是十分的正确。另一方面也为了提升下面的fullTryAcquireShared中使用这个HoldCounter的性能（它也是直接去cachedHoldCounter中拿，而不是去ThreadLocal中找）

可以看出在一轮中第一个获取读锁的线程被特殊对待了，它并不是用HoldCounter来计数的，而是直接使用Sync类的int型成员变量firstReaderHoldCount，它的引用也被直接保存在了Sync类的Thread类型的成员变量firstReader中，用于判断它的重入，这个主要是为了优化只有一个线程争抢读锁的情况。

从上面的这些优化可以看出，作者为了提升性能，是能不进入ThreadLocal就不进入ThreadLocal，可见作者对ThreadLocal的性能是有顾虑的（当然ThreadLocal相比其他并发控制工具，性能还是很好的），我打算单独写篇文章来探讨ThreadLocal的性能。

## ④

如果在②处的判断没能成功通过，则线程还有机会在fullTryAcquireShared方法中通过自旋来获得锁，如果这次机会依旧没能抓住，就只能返回-1了（抢锁失败）。

## ⑤

这一部分其实就是进行类似于①②中进行的判断，如果发现不通过，直接返回-1，其中对锁的重入进行了很多判断：

```java
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)   //存在写锁且持有锁的不为当前线程则返回-1
                        return -1;
                    //锁降级的情况

                } else if (readerShouldBlock()) {//没有线程持有写锁的情况
                    
                    if (firstReader == current) {
                      
                    } else {
                        //非重入线程运行到这里肯定会抢锁失败
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            //如果发现缓存的cachedHoldCounter不是当前线程的重入计数器
                            if (rh == null || rh.tid != current.getId()) {
                                rh = readHolds.get();
                                if (rh.count == 0)  //线程非重入
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)   //线程非重入
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
```

与①②判断中不同的是，如果这里发现是锁降级，即`exclusiveCount(c) != 0 && getExclusiveOwnerThread() == current`，则立即进入下一步获得锁，不会再进行readerShouldBlock的判断，如果单步执行锁降级的代码，会发现当写锁重入读锁的时候，直到`fullTryAcquireShared`方法才能成功获得读锁，通过阅读代码可以确定，写锁重入读锁时，第一次`tryAcquireShared`应该都能成功，所以重入读锁并不会在CLH队列中形成一个新的节点，这么做主要是为了防止锁降级的过程中发生死锁。

## ⑥

⑥基本上就是③的翻版：

```java
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {   //首次获取读锁
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {   //第一个得到读锁的线程重入读锁
                        firstReaderHoldCount++;
                    } else {
                        //第一个得到读锁的线程以外的线程获得读锁
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != current.getId())
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
```

如果CAS改变state的读状态成功了，则进行对Sync对象进行一些状态的变更并且返回-1。关于状态的变更，和③一样，主要就是要特殊考虑第一个获取到读锁的线程，以及缓存最近一次获得读锁的线程的重入计数器cachedHoldCounter 。


## tryReleaseShared

理解了tryAcquireShared之后，tryReleaseShared就易如反掌了，无非就是先将重入计数器减去1（因为第一个获取读锁的线程是被“特殊对待”的，所以可能要多写一点代码进行判断），然后使用CAS减去读状态的值，代码如下：

```java
        protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            //如果释放读锁的线程为第一个获取读锁的线程
            //第一个线程单独处理
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else { //处理第一个获取读锁的线程以外获取读锁的线程
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != current.getId())
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            //CAS循环更新状态
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
```


# End
---


