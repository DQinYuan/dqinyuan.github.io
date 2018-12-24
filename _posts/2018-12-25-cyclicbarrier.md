---
layout: post
title: 图解java.util.concurrent源码 （六）CyclicBarrier (循环栅栏)
date: 2018-12-25
categories: java
tags: java 并发
cover: https://upload-images.jianshu.io/upload_images/10192684-a81f413143f9cded.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
---

# 引言
---

上一篇文章提到，CountDownLatch不支持重置计数，如果你有反复重置计数的需求的话，最好使用CyclicBarrier。

CyclicBarrier的中文名叫做"循环栅栏"，能够让n个线程都到达同步点之后再让他们开始运行，之后CyclicBarrier就会重新计数，这个过程可以反复进行，甚至还可以在到达同步点与重新运行之间插入一段代码（叫做barrierAction）。

# Demo
---

文章重点讲的是实现，但是出于完整性考虑还是给个Demo，Demo摘自《实战Java高并发程序设计》。

场景：有10个士兵（其实就是10个线程），他们要先进行集合，集合完毕后会打印"司令：[士兵10个，集合完毕!]"（其实这就是一个barrierAction），然后开始各自的工作，工作完毕后士兵们再集合起来，此时会打印"司令：[士兵10个, 任务完成!]"（同理,这也是一个barrierAction）。

代码实现如下：

```java
import java.util.Random;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierDemo {

    public static class Soldier implements Runnable{

        private String soldier;

        private final CyclicBarrier cyclic;

        public Soldier(CyclicBarrier cyclic, String soldier) {
            this.soldier = soldier;
            this.cyclic = cyclic;
        }

        @Override
        public void run() {
            try {
                //等待士兵到齐
                cyclic.await();
                //士兵开始做各自的工作
                doWork();
                //等待所有士兵完成任务
                cyclic.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }

        void doWork(){
            try {
                Thread.sleep(Math.abs(new Random().nextInt() % 10000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(soldier + " 任务完成");
        }
    }

    /**
     * 这个类用于barrierAction
     */
    public static class BarrierRun implements Runnable{

        boolean flag;

        int N;

        public BarrierRun(boolean flag, int n) {
            this.flag = flag;
            N = n;
        }

        @Override
        public void run() {
            if ( flag ){
                System.out.println("司令：[士兵" + N + "个, 任务完成!]");
            } else {
                System.out.println("司令：[士兵" + N + "个，集合完毕!]");
                flag = true;
            }
        }
    }

    public static void main(String[] args) {
        final int N = 10;
        Thread[] allSoldier = new Thread[N];
        boolean flag = false;
        /**
         * 插入了BarrierRun作为barrierAction
         */
        CyclicBarrier cyclic = new CyclicBarrier(N, new BarrierRun(flag, N));

        for ( int i = 0; i < N; i++ ){
            System.out.println("士兵 " + i + " 报道");
            allSoldier[i] = new Thread(new Soldier(cyclic, "士兵 " + i));
            allSoldier[i].start();
        }
    }

}

```

图示如下(图中下标号对应着下面文字描述的标号)：

![CyclicBarrier运作过程](https://upload-images.jianshu.io/upload_images/10192684-a81f413143f9cded.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 1. 士兵陆续前来集合
 2. 士兵集合完毕
 3. barrierAction1: 打印"司令：[士兵10个，集合完毕!]"
 4. 士兵陆续完成任务
 5. 所有士兵的任务都执行完毕
 6. barrierAction2: 打印"司令：[士兵10个, 任务完成!]"

运行打印的结果如下：

```c-like
士兵 0 报道
士兵 1 报道
士兵 2 报道
士兵 3 报道
士兵 4 报道
士兵 5 报道
士兵 6 报道
士兵 7 报道
士兵 8 报道
士兵 9 报道
司令：[士兵10个，集合完毕!]
士兵 7 任务完成
士兵 2 任务完成
士兵 9 任务完成
士兵 0 任务完成
士兵 3 任务完成
士兵 5 任务完成
士兵 4 任务完成
士兵 1 任务完成
士兵 6 任务完成
士兵 8 任务完成
司令：[士兵10个, 任务完成!]
```


你可能会疑惑这个barrierAction是由哪个线程负责执行的呢？从图中可以看出barrierAction每次都是由一个线程执行的，而这个线程一般就是最后到达的那个线程，之后我也会通过源码分析得出这个结论。

从上面的demo看出CyclicBarrier的核心方法就只有一个`await`，它会抛出两个异常，`InterruptedException `和`BrokenBarrierException`。`InterruptedException`显然是当前线程等待的过程被中断而抛出的异常，而这些要集合的线程有一个线程被中断就会导致线程永远都无法集齐，导致“栅栏损坏”，剩下的线程就会抛出`BrokenBarrierException`异常。

想要看到“栅栏损坏”的现象只要把main方法改成如下即可：

```java
    public static void main(String[] args) {
        final int N = 10;
        Thread[] allSoldier = new Thread[N];
        boolean flag = false;
        /**
         * 插入了BarrierRun作为barrierAction
         */
        CyclicBarrier cyclic = new CyclicBarrier(N, new BarrierRun(flag, N));

        for ( int i = 0; i < N; i++ ){
            System.out.println("士兵 " + i + " 报道");
            allSoldier[i] = new Thread(new Soldier(cyclic, "士兵 " + i));
            allSoldier[i].start();
            if ( i == 5 ){
                allSoldier[0].interrupt();
            }
        }
    }
```

然后你会得到1个`InterruptedException`和9个`BrokenBarrierException`。

# 栅栏损坏
---

其实除了上面说的情况会发生“栅栏损坏”，文档中还提到了好几种会发生的情况，如下：

 - 有一个线程发生中断(就是上一小节提到的那个情况)或者超时，而当前线程正在等待（await），则当前线程会抛出`BrokenBarrierException`
 - 该CyclicBarrier对象被调用了reset方法
 - 该CyclicBarrier对象被调用await时，状态已经是"broken"了
 - barrierAction抛出了未捕获的异常


# 实现分析
---

与CountDownLatch不同，CyclicBarrier不是基于AQS实现，而是应用ReentrantLock实现的，它的同步靠的是两个成员变量（分别是一个ReentrantLock以及从中引申出的Condition）：

```java
    /** The lock for guarding barrier entry */
    private final ReentrantLock lock = new ReentrantLock();
    /** Condition to wait on until tripped */
    private final Condition trip = lock.newCondition();
```


我们先从构造函数看起：

```java
    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }
```

从这里看出几个关键成员变量的含义，parties代表循环栅栏每次要等待多少个线程，count则是用于倒计数用的计数器，而barrierCommand就是barrierAction了。

下面看CyclicBarrier唯一的核心方法await：

```java
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen;
        }
    }
```

await方法返回的数值是线程的达到次序，对于第一个到达的线程会返回（总的数值 - 1），而最后一个到达的线程会返回0。

发现它其实调用的是dowait方法。

dowait方法整个被一把ReentrantLock给锁住了：

```java
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            ...
        } finally {
            lock.unlock();
        }
    }
```

被锁住的这段代码看起来长，其实总结起来由以下步骤组成：

![dowait流程](https://upload-images.jianshu.io/upload_images/10192684-f590dcfaa52ed4e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我在下面的代码中标出了相应步骤(图中的①对应下面注释中的"一"，②对应"二"，以此类推)：

```java
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

           //一：倒计数
           int index = --count;
           //二：所有线程都已就位?
           //index为0说明所有线程都已就位
           if (index == 0) {  // tripped
               boolean ranAction = false;
               try {
                   final Runnable command = barrierCommand;
                   //三：最后一个到达的线程顺手执行barrierAction
                   if (command != null)
                       command.run();
                   ranAction = true;
                   //四：nextGeneration方法会唤醒所有线程并且更新generation进入下一代
                   nextGeneration();
                   //最后一个到达的线程返回0
                   return 0;
               } finally {
                   //如果barrierAction出现异常则将该循环栅栏损坏
                   if (!ranAction)
                       breakBarrier();
               }
           }

            // 五：利用Condition进行等待
            for (;;) {
                try {
                    if (!timed)   //无超时等待
                        trip.await();
                    else if (nanos > 0L)   //有超时等待
                        nanos = trip.awaitNanos(nanos);//返回剩余的时间
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    //超时则破坏循环栅栏
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }

```

下面分别对每一块进行讲解。

### ①之前

在我列出的关键步骤①之前还有几行代码，我们先来看一看：

```java
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }
```

CyclicBarrier对象有一个Generation内部类，其唯一的作用是标记这一代是否出现了"栅栏损坏"：

```java
    private static class Generation {
        //为true则表示这一代发生了栅栏损坏
        boolean broken = false;
    }
```

所有的打破栅栏的控制流都会调用`breakBarrier`方法：

```java
    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }
```

可以看到它把generation的broker标志设置为了true，然后重置了count计数器，最后使用了trip这个Condition对象唤醒了所有的线程。generation成员变量会在每次线程到齐之后换一个新的（我称之为"换代"，分析到后面的代码时再说）。

知道了这些，刚开始给出的那几行代码就很好理解了，如果线程刚到这里就发现栅栏在这一代以及损坏（`if (g.broken)`），则直接抛出栅栏损坏异常（`throw new BrokenBarrierException();`）。

如果发现线程在之前以及被中断了（`if (Thread.interrupted()) {`），则立即损坏栅栏（`breakBarrier();`）并抛出`InterruptedException`。


### ①: 倒计数

这个很简单，就是将count成员变量减1，然后再保存到局部变量index中，等到方法返回时，作为返回值。

### ②：所有线程都已就位？

这个也很简单，当count减到0了，则说明所有线程都就位了，否则还需要等。

### ③④：顺手执行barrierAction，唤醒其他线程并进行下一代

```java
           //二：所有线程都已就位?
           //index为0说明所有线程都已就位
           if (index == 0) {  // tripped
               boolean ranAction = false;
               try {
                   final Runnable command = barrierCommand;
                   //三：最后一个到达的线程顺手执行barrierAction
                   if (command != null)
                       command.run();
                   ranAction = true;
                   //四：nextGeneration方法会唤醒所有线程并且更新generation进入下一代
                   nextGeneration();
                   //最后一个到达的线程返回0
                   return 0;
               } finally {
                   //如果barrierAction出现异常则将该循环栅栏损坏
                   if (!ranAction)
                       breakBarrier();
               }
           }
```

最后一个到达的线程发现count已经减到0了，如果发现设置了barrierAction的话，则顺手将其执行，紧接着调用nextGeneration方法将整个CyclicBarrier对象的状态进行重置，准备迎接下一轮循环，nextGeneration方法的内容如下：

```java
    private void nextGeneration() {
        // signal completion of last generation
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
    }
```

trip就是从lock中衍生出来的Condition对象，这里调用`signalAll`方法将所有阻塞住的线程都唤醒了（这些线程都阻塞在⑤中），将count重新置位倒计数的总数parties，之后将generation成员变量换了个新的（称之为"换代"）。

最后的`finally`代码块中，会通过ranAction标志检测barrierAction是否成功执行，如果未能成功执行，则还是会调用`breakBarrier`方法破坏掉栅栏。

从上面的代码可以看出，CyclicBarrier一旦损坏掉就不会自动回复了，需要手工调用CyclicBarrier对象的`reset`方法来开启新的一代。

### ⑤：利用Condition进行等待

除了最后一个到达的线程，其他线程都会进入这一段代码进行等待，核心就是使用`trip`这个Condition对象的`await`方法阻塞住：

```java
                    if (!timed)   //无超时等待
                        trip.await();
                    else if (nanos > 0L)   //有超时等待
                        nanos = trip.awaitNanos(nanos);//返回剩余的时间
```

如果await的过程中发生了中断，则在catch代码块中破坏栅栏（`breakBarrier`），如果发现已经换代或者栅栏已经损坏，则重置当前线程的中断标志位（`Thread.currentThread().interrupt();`），供下一轮使用：

```java
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

```

剩下的三个判断就是线程被唤醒之后的处理了：

```java
                //唤醒之后发现栅栏已经损坏,则抛出异常
                if (g.broken)
                    throw new BrokenBarrierException();

                //栅栏没有损坏而且换代了,说明这一代顺利结束，await方法返回
                if (g != generation)
                    return index;

                //超时
                if (timed && nanos <= 0L) {
                    //超时则破坏循环栅栏
                    breakBarrier();
                    throw new TimeoutException();
                }
```

这里注意到一个比较有趣的地方就是，假如nanos大于0的情况下则会继续Condition等待的循环，nanos怎么会大于0呢？查阅了`awaitNanos`的文档发现它的返回值的含义是**线程剩余的需要等待的时间**，为了让`awaitNanos`能够真正地等待你所指定的时间，推荐的写法是：

```java
 boolean aMethod(long timeout, TimeUnit unit) {
   long nanos = unit.toNanos(timeout);
   lock.lock();
   try {
     while (!conditionBeingWaitedFor()) {
       if (nanos <= 0L)
         return false;
       nanos = theCondition.awaitNanos(nanos);
     }
     // ...
   } finally {
     lock.unlock();
   }
 }
```

其实CyclicBarrier中的写法就是这种推荐写法的变种。

当时我就很疑惑，为什么这个`awaitNanos`方法这么不靠谱呢？我都让它等待`nanos`之后再超时，它却有可能中途就原因不明地给我返回了。

回忆一下AQS中`awaitNanos`的实现，它阻塞靠的是`LockSupport.parkNanos`方法，而`LockSupport`的文档明确说了，它的`parkNanos`会出现**"伪唤醒"**的问题（也就是原因不明地返回），而且因为`parkNanos`是个返回值为`void`的方法，它甚至不会告诉你剩余的时间是多少，AQS对其额外封装了剩余的等待时间，也算是比较友好了。

# 总结
---

相比JUC中的其他类，CyclicBarrier的实现属于比较接地气的，基于ReentrantLock实现了自己的功能，可以学习它对于ReentrantLock的应用


# 参考文献
---
《实战Java高并发程序设计》(葛一鸣 郭超 著)
