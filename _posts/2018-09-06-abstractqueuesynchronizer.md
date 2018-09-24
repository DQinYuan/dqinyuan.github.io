---
layout: post
title: 图解java.util.concurrent源码（一）AbstractQueuedSynchronizer（AQS）
date: 2018-09-06
categories: java
tags: java 并发
---

# 引言
---
这个系列文章打算用图解的方式记录了自己阅读concurrent包的中一些类的大概流程，加深印象。
# JDK版本
---
我这里依据的JDK版本如下：
```c-like
java version "1.8.0_73"
Java(TM) SE Runtime Environment (build 1.8.0_73-b02)
Java HotSpot(TM) 64-Bit Server VM (build 25.73-b02, mixed mode)
```
如果你的版本和我不同，看到的源码可能有细微的不同。

# 什么是AbstractQueuedSynchronizer
---
concurrent包下的很多类都有一个叫做Sync的内部类（比如ReentrantLock，ThreadPoolExecutor等），并且很多功能会委托给这个内部类，而这个内部类实现了AbstractQueuedSynchronizer（下面简称AQS）。
# AQS的功能
---
按照官方文档的说法，通过AQS可以很方便的实现一个自定义的同步器，子类只需要通过重写以下方法来控制AQS内部的一个叫做state的同步变量：
```java
//独占模式获取锁与释放锁
//返回值表示获取锁是否成功
protected boolean tryAcquire(int arg)
//返回值表示释放锁是否成功
protected boolean tryRelease(int arg)
//共享模式获取锁与释放锁
//返回值表示获得锁后还剩余的许可数量
protected int tryAcquireShared(int arg)
//返回值表示释放锁是否成功
protected boolean tryReleaseShared(int arg)
```

独占模式与共享模式的含义是：
 - 独占模式：资源是互斥的，一次只能一个线程获取锁
 - 共享模式：资源一次可以由n个线程同时使用（n有限）


在重写这些方法时，如果想要使用state同步变量，必须使用AQS内部提供的以下方法来控制：
```java
protected final int getState()
protected final void setState(int newState)
protected final boolean compareAndSetState(int expect, int update)
```

一个AQS的使用示例如下（这里用AQS实现一个简单的不可重入锁）：
```java
public class SimpleLock extends AbstractQueuedSynchronizer {

    @Override
    protected boolean tryAcquire(int unused) {
       //使用compareAndSetState控制AQS中的同步变量
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    @Override
    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        //使用setState控制AQS中的同步变量
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    /**
    *发现线程是顺序获得锁的
    * 因为AQS是基于CLH锁的一个变种实现的FIFO调度
    */
    public static void main(String[] args) throws InterruptedException {
        final SimpleLock lock = new SimpleLock();
        lock.lock();
        for (int i = 0; i < 10; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    lock.lock();
                    System.out.println(Thread.currentThread().getId() + " acquired the lock!");
                    lock.unlock();
                }
            }).start();
            // 简单的让线程按照for循环的顺序阻塞在lock上
            //目的是让线程顺序启动
            Thread.sleep(100);
        }

        System.out.println("main thread unlock!");
        lock.unlock();
    }
```

其实这个基本上就是ThreadPoolExecutor的内部类Worker对AQS的实现，后面的文章我再说。

# CLH锁
---
AQS的原理是CLH锁的一个变种，具体怎么变种的后文再说，这里先说一下什么是CLH锁。
讲之前先推荐一本书--《The Art of Multiprocessor Programming
》，这本书对并发的概念和各种锁的设计思想介绍得特别清楚，目前没发现中文版，文末的参考文献我附了一个英文版的下载链接。书的7.5.2节介绍了CLH锁的设计思想，我这里就从书中摘抄一下只言片语简要介绍。
 CLH锁其实是自旋锁的一种改良，与一般的自旋锁不同，一般的自旋锁会将并发所有的竞争集中在一个标志位里，而CLH锁将竞争资源的线程排成一个队列，每个线程只在前一个线程的标志位上进行自旋，当头节点释放锁时，将自己的标志位置为false，这样后继线程在自旋时发现标志位变为false后，便能获得锁进入临界区。
CLH队列大概看起来像下面这样：
![CLH锁原理](https://upload-images.jianshu.io/upload_images/10192684-39bbdb7cede64b0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

CLH锁的这种设计思想叫做Local Spin，它能够最大化地减少CPU缓存的失效次数。现在的多核CPU一般每一个物理核都会有自己的缓存，假如是普通的自旋锁，所有CPU核都自旋在一个标志位上，因为这个标志位竞争非常激烈，所以标志位经常会变化，每当标志位变化时，所有CPU的缓存就会失效，这样显然无法最大程度上利用CPU的缓存，而在CLH锁的设计中，每个线程只需要在自己的前继的标志位上自旋即可，而前继的标志位仅仅在前继释放锁的时候会发生变化，这样每个CPU核就可以一直在自己的本地缓存上自旋（所以称之为Local Spin）而不会出现频繁的缓存失效，减少了缓存失效，锁算法效率自然就提高了。

一个最标准的CLH锁的实现如下：
```java
public class CLHLock implements Lock {
    AtomicReference<QNode> tail = new AtomicReference<QNode>(new QNode());
    ThreadLocal<QNode> myPred;//代表前继的节点
    ThreadLocal<QNode> myNode;//代表当前线程的节点
    public CLHLock() {
        tail = new AtomicReference<QNode>(new QNode());
        myNode = new ThreadLocal<QNode>() {
			protected QNode initialValue() {
				return new QNode();
			}
		};
		myPred = new ThreadLocal<QNode>() {
			protected QNode initialValue() {
				return null;
			}
		};
	}
	
	public void lock() {
		QNode qnode = myNode.get();
		qnode.locked = true;
		QNode pred = tail.getAndSet(qnode);
		myPred.set(pred);
        //在前继节点的标志位上自旋
		while (pred.locked) {}
	}
	
	public void unlock() {
		QNode qnode = myNode.get();
        //将当前线程节点的标志位置为false
		qnode.locked = false;
        //此时代表前继节点的QNode对象已经没有用了,这里将其复用
		myNode.set(myPred.get());
	}
}
```

CLH锁在大多数情况下表现都很优异，书中只给了一处例外，就是CLH锁不适合用在NUMA体系结构的计算机上，在NUMA体系结构的计算机上则应该使用另外一种同样是基于队列的锁方法--MCS锁，具体什么是MCS锁这里就不展开说，有兴趣的可以去看我推荐的那本书的7.5.3节。

至于什么是NUMA体系结构的计算机，可以看一看这篇文章[http://www.cnblogs.com/yubo/archive/2010/04/23/1718810.html](http://www.cnblogs.com/yubo/archive/2010/04/23/1718810.html)

因为与AQS无关，我这里就不再多说了。

# AQS原理概览
---
在真正开始阅读源码之前，我先用简要地说明一下AQS的原理。
AQS维护着两个队列，一个是由AQS类维护的CLH队列（用于运行CLH算法），另一个是由AQS的内部类ConditionObject维护的Condition队列（用于支持线程间的同步，提供await,signal,signalAll方法）。

AQS中维护的CLH队列看起来大概像这样：
![AQS中的CLH队列](https://upload-images.jianshu.io/upload_images/10192684-998aa2b8c1e7801f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

具体运作看接下来的源码详解。

# AQS源码图解
---
有了CLH锁相关的知识后，就可以来看一看AQS到底是怎么应用这一优秀的锁算法的了。
## Node内部类
前面说过，CLH锁是基于队列的，队列中每个节点对应着一个等待资源的线程，在AQS中这个节点对用这一个叫做Node的内部类来表示，我列举一下它比较中要的几个字段：
 - waitStatus: 等待状态，有以下几种取值

```java
//代表线程已经被取消
static final int CANCELLED = 1;

//代表后续节点需要唤醒
static final int SIGNAL = -1;

//代表线程在condition queue中，等待某一条件
static final int CONDITION = -2;

//代表后续结点会传播唤醒的操作，共享模式下起作用
static final int PROPAGATE = -3;
```

 - prev: CLH队列的前继
 - next: CLH队列的后继
 - nextWaiter: Condition队列的后继
 - thread: 这个节点所代表的线程

从这几个字段可以看出，AQS中维护着两个队列（两个队列都是由Node组成），一个队列就是CLH锁算法中的那个队列（我将其称之为CLH队列），另一个是Condition队列（下文再讲Condition队列是用来做什么的）。

## acquire方法
acquire方法是在独占模式下用于获取锁的，它的主体逻辑我梳理了一下，如下（只梳理了主题逻辑，所以代码中一些我认为不重要的细节就忽略了）：
![acquire方法流程图](https://upload-images.jianshu.io/upload_images/10192684-b85d2598dbf30a0e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



图中红色的节点代表开始和终止的节点，图中节点编号对应代码如下。
### ①
```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
这个节点对应着acquireQueued方法的第一个参数中对addWaiter的调用。
addWaiter方法会为当前线程建立一个Node节点，并将节点加入CLH队列后返回。
从上面这段代码我们也可以看出，在开始真正的CLH算法（即acquireQueued方法）之前，会先尝试一下获得锁（即由子类重写的tryAcquire方法），这样在竞争较小的情况下能够提升程序的性能。

### ②
①结束之后，剩下的部分便全部在acquireQueued方法中进行，这个方法的代码如下：
```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
重点在这个for循环
对应着②的是for循环的前两句：
```java
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
```
如果发现了自己的前继已经是头节点了的话，则说明此时有获得锁的可能，就会调用tryAcquire进行尝试。

### ③
③是for循环的终结状态，当前面的tryAcquire获取锁成功时会执行如下代码（863~868行）：
```java
                //这里p是前继节点
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
```
可以看到这里把自己对应的Node设置成了头结点，抛弃掉了原本的头节点（即前继出队）。

### ④
当尝试获取锁失败时，当前线程就要考虑一下是否要将自己阻塞了，这个逻辑位于shouldParkAfterFailedAcquire方法中，代码如下：
```java
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            return true;
        if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
                //waitStatus大于0的情况只有CANNCELLED
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
④就对应这个该方法的第二行，判断前继的waitStatus是SIGNAL（说明前继已经准备好唤醒后继节点）。

### ⑦
这里先将⑦，从⑥中的代码可以看出，如果前继的waitStatus是SIGNAL，则shouldParkAfterFailedAcquire直接放回true，然后参考一下上层的acquireQueued方法：
```java
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
```
发现shouldParkAfterFailedAcquire返回true的话就会执行parkAndCheckInterrupt方法，parkAndCheckInterrupt方法的内容如下：
```java
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
发现AQS其实是调用concurrent包下的LockSupport来阻塞线程的。

### ⑥
如果发现前继被CANCELLED了，则会跳过前继，一直找到第一个没有被CANCELLED的节点作为自己的前继，代码如下(shouldParkAfterFailedAcquire方法的第二个if判断)：
```java
        if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
                //waitStatus大于0的情况只有CANNCELLED
            } while (pred.waitStatus > 0);
            pred.next = node;
        } 
```

### ⑤
上面的if语句对应else就是⑤的代码：
```java
        } else {
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
```
当前继节点的waitStatus为0, PROPAGATE或者CONDITION时（其实这里不可能为CONDITION，这个是留在Condition队列中使用的waitStatus，acquire方法不会涉及Condition队列），则将前继节点的waitStatus设置为SIGNAL。

### 总体流程

从我画得流程图中可以看出，acquire方法在几次尝试获得锁失败后成功地将前继线程的waitStatus设置为SIGNAL，然后阻塞自己。之后在某个时刻会被前继线程唤醒，然后有经过几次争抢后可能会成功地获得锁。

## release方法
相比上面的acquire方法，release方法可以说是非常的简单，它做的就是如果tryRelease成功，就将头结点的下一个节点对应的线程唤醒：
```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

unparkSuccessor方法会将h节点的后继唤醒，点开这个方法会发现更多细节，比如，如果发现h的后继节点为null或者状态是CANCELLED时，会找出离tail最远（或者说离h节点最近）的一个非CANCELLED节点唤醒，代码如下：
```java
    private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);


        Node s = node.next;
        /*
           如果发现node的后继节点为null或者状态是CANCELLED时，
           会找出离tail最远（或者说离node节点最近）的一个非CANCELLED节点唤醒
         */
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
可以看出这里也是依靠LockSupport来唤醒线程的。
后继线程被唤醒之后就会从前面我画的acquire方法的流程图中的节点⑦开始不断地尝试获得锁，直到成功。

### AQS中对CLH算法的实现与标准的CLH算法有什么异同？
到这里已经可以解答这个问题了。AQS到底在哪些地方"变种"CLH锁算法？
 1.  CLH是一种自旋锁算法（在得到锁之前会不停地自旋），而AQS会在几次自旋失败后就将线程阻塞，这是为了避免不必要地占用CPU；
 2. CLH是自旋在前继节点的标志位上的，而AQS是自旋在`p == head`上面（即不停地判断前继节点是否是头节点），只有在发现前继节点是头节点时，才会通过`tryAcquire`尝试获得锁，这里有一个比较另我困惑的地方，就是head是一个volatile的全局引用，这么做的话显然违背了CLH锁的Local Spin的思想，具体原因未知，可能是因为AQS最初就是被设计为阻塞的同步器而不是自旋锁吧。

## acquireShared方法
点进这个方法，发现其逻辑和acquire基本一样，唯独不同的地方如下：
```java
    private void doAcquireShared(int arg) {
        //共享模式入队
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        //这里不同
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
第一行的addWaiter会将节点标记为共享模式入队（这个标记其实就是在Node的nextWaiter属性上添加一个Node.SHARED，在CLH队列中，nextWaiter属性没有用，所以这里就暂时拿来标记，isShared方法会用这个标记来判断节点是否为共享模式）。
虽然写法和acquire不太一样，但是可以看出逻辑基本相同，唯一值得注意的地方是，acquire方法在tryAcquire成功时，直接setHead将自己置为CLH队列的队头，而这里调用了一个叫做setHeadAndPropagate的方法，虽然名字看起来差不多，但是逻辑却很不相同，点开来看看：
```java
    /**
     * @param node      the node
     * @param propagate 剩余许可数量
     */
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);

        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```

上面的代码除了将自己置为头节点外，还会继续尝试唤醒后继节点（doReleaseShared），让他们也来尝试争抢锁。
doReleaseShared的代码如下：
```java
    private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```
可以看出这个方法会让head的waitStatus产生如下变化：
![共享模式waitStatus的变化过程](https://upload-images.jianshu.io/upload_images/10192684-f4a206b2145c9ed5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## releaseShared方法

主要就是调用上面提到的doReleaseShared方法：
```java
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

从这里我们可以看出doReleaseShared方法会在线程获得锁和释放锁时分别调用一次，所以在共享模式下一个线程对应的节点比较常见的状态转移大约是`0(新建节点时) -> SIGNAL -> 0 -> PROPAGATE`

## ConditionObject内部类

虽然加锁和释放锁的代码都讲完了，但是AQS远没有那么简单。
仔细看一下源码，发现AQS还有一个叫做ConditionObject的内部类，这个类是用来干嘛的呢？
在使用conncurrent包中的锁（比如`ReentrantLock`）的时候，我们一般会使用`lock.newCondition()`方法返回一个Condition对象来对争抢锁的线程进行同步。
看一下ReentrantLock的newCondition方法的代码：
```java
    public Condition newCondition() {
        return sync.newCondition();
    }
```
发现是委托给内部类Sync的一个实例sync的newCondition方法，在点进去看看Sync的newCondition方法，内容如下：
```java
        final ConditionObject newCondition() {
            return new ConditionObject();
        }
```
这里出现了ConditionObject，Sync继承了AQS，这里的ConditionObject就是AQS中的内部类ConditionObject，这个类几乎不需要经过任何修改就可以直接用来同步，感觉很神奇。
其实ConditionObject内部又维护了一个队列，我称之为Condition队列，这个队列同样是由Node类的实例组成。
我们来看一下它的三个重要方法，分别是：
 - await
 - signal
 - signalAll


## await方法

await方法的主体逻辑如下：

![await方法](https://upload-images.jianshu.io/upload_images/10192684-10fc4e95f6c089dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```java
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            //流程图一
            Node node = addConditionWaiter();
            //二
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            //三
            while (!isOnSyncQueue(node)) {
                //四
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            //五
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

### ①
对应着addConditionWaiter方法的调用，这个ConditionObject实例自己的方法。从addConditionWaiter方法可以看出，Node类的nextWaiter字段其实是用来存放Condition队列的后继的，要和next字段（用来存放CLH队列后继）进行区分。

| 字段    | 含义    |  
| --------   | :-----   | 
| next        | CLH队列中的后继      |   
| nextWaiter        | Condition队列中的后继，如果Node节点在CLH队列中，这个字段也可能作为共享模式（Node.SHARED）的标记      |  

```java
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```

### ②
对应着fullyRelease方法的调用，fullyRelease会先保存当前同步变量（state），然后通过之前讲过的release方法将其全部释放掉，最后将其保存的同步变量（state）返回给上层方法，留复原的时候用，代码如下：
```java
    final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```

### ③

这一步其实就是while循环的判断条件isOnSyncQueue方法：
```java
    final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;
        return findNodeFromTail(node);
    }
```
这个方法其实就是在判断node节点是否在CLH队列中，这个node就是在第①步中创建并加入Condition队列的节点。
可能会疑惑，明明是在Condition队列中的节点，怎么又突然跑到CLH队列中呢？其实是下文中的signal方法搞的鬼。

### ④
这一步很明显就是while循环中的`LockSupport.park(this);`，这里也是利用LockSupport阻塞线程的。
这里需要注意的一个细节是下面两句的中断检查：
```java
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
```
点开checkInterruptWhileWaiting方法：
```java
        /**
         * Checks for interrupt, returning THROW_IE if interrupted
         * before signalled, REINTERRUPT if after signalled, or
         * 0 if not interrupted.
         */
        private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }

```
英文注释中已经解释了这个方法的含义：如果中断发生在signal之前，则在await最后会抛出InterruptedException异常（这里标记为THROW_IE）；如果中断发生在signale之后，则选择在最后重置中断位（标记为REINTERRUPT）。标记的处理在await方法的最后一句reportInterruptAfterWait调用里面进行，reportInterruptAfterWait内容如下：
```java
        private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }
```
含义非常显而易见。
为什么要做这两种不同的中断处理呢？我觉得是为了方便上层应用区分：线程从await方法中苏醒究竟是因为中断（THROW_IE）还是因为被其他线程signal（REINTERRUPT）。

| 标记    | 行为   |含义|  
| --------   | :-----   | :-----   | 
| THROW_IE       | await方法的最后抛出InterruptedException异常      |  线程苏醒是由中断引起的 |
| REINTERRUPT| await方法的最后重置中断标志位   | 线程苏醒是由其他线程调用signal方法引起的|  

再回到checkInterruptWhileWaiting方法，这个方法中处理判断要怎么处理中断以外，transferAfterCancelledWait调用还干了其他事情，代码如下：
```java
    final boolean transferAfterCancelledWait(Node node) {
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
            enq(node);
            return true;
        }
        /*
         * If we lost out to a signal(), then we can't proceed
         * until it finishes its enq().  Cancelling during an
         * incomplete transfer is both rare and transient, so just
         * spin.
         */
        while (!isOnSyncQueue(node))
            Thread.yield();
        return false;
    }
```
这个方法一定会等到节点加入CLH队列才会返回，而且按照英文注释，下面那种自旋的情况只有在release方法将节点从Condition队列搬运到CLH队列的工程中才会发生，发生的可能性是很低的，所以我们可以认为当线程被中断后，它就会立刻将自己加入CLH队列。

### ⑤
⑤主要就是对我之前讲过的acquireQueued方法的调用，里面的过程见我对acquire方法的讲解，线程会在这个方法里反复尝试，直到获得锁才会退出该方法（即便遇到了中断也直到获得锁才会退出，这时acquireQueued返回true）。

## signal方法

主体逻辑的流程图如下：

![signal方法](https://upload-images.jianshu.io/upload_images/10192684-10a9c8bb97af4472.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### ①  ②

signal方法的代码如下：
```java
        public final void signal() {
            //流程图节点一
            if (!isHeldExclusively())
                //二
                throw new IllegalMonitorStateException();
            //剩下的步骤（三,四,五,六,七）
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
```

①②两步对应着signal方法的前两句，非常显然，就不多说了。

### ③
接下来的步骤的关键是doSignal方法：
```java
        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
```
第二行的`firstWaiter = first.nextWaiter`会获取Condition队列的下一个节点，这个节点被获取过之后就会被从Condition队列中删除。


### ④
④其实就是`while`语句中的第一个判断条件，第二个判断条件很好懂，就是如果一直到队列尾部都没有找到合适的节点，循环就结束。
这个循环中需要注意一点的是，虽然每次一进来就会获取下一个节点（将Condition队列队头(firstWaiter)设置为下一个节点 ），但是传进while的第一个判断条件transferForSignal方法的参数其实是前一个节点（first），第二个判断条件用的才是firstWaiter节点。
第一个判断条件的含义需要点开transferForSignal方法才能明白：
```java
    final boolean transferForSignal(Node node) {
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```
只有在第一行的CAS操作失败时才会返回false，个人认为在这里CAS操作失败的唯一可能就是node的waitStatus不是CONDITION，所以我认为第一个判断条件的含义就是判断节点的waitStatus是否是CONDITION。

### ⑤
从transferForSignal方法中可以看出在CAS操作成功后，首先就会调用`enq`方法将node节点加入CLH队列（node节点在前面的while循环中已经从Condition队列删除了，所以node节点同时只会在一个队列），enq方法同时会返回node节点的前继。在这里我们也可以看出，之所以Condition队列和CLH队列都采用Node类作为节点的原因就是为了方便将节点从Condition队列搬运到CLH队列。

### ⑥ ⑦
transferForSignal的最后几句干的就是这件事情：
```java
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
```
调用signal方法的线程先企图帮助node节点对应的等待线程将它的CLH队列的前继的waitStatus设置为SIGNAL，如果不成功的话才将等待线程唤醒，让其自行设置。
之所以要进行这么一次尝试，是为了减少线程切换的开销，尽量在当前线程把事情都做掉，就不再麻烦等待线程了，等到有资源的时候，自然会有它CLH队列的前继来将其唤醒。

### signalAll方法

它与signal唯一的不同就在于，将transferForSignal方法加入了循环体，并且将while循环的判断条件改成了`first != null`。

```java
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);
```
也就是说它会遍历完Condition队列将他们全部加入CLH队列。
