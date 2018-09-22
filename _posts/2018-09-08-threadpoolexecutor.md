---
layout: post
title: 图解java.util.concurrent源码（二）ThreadPoolExecutor
date: 2018-09-08
categories: java
tags: java 并发
---

# JDK版本
---
我这里依据的JDK版本如下：
```c-like
java version "1.8.0_73"
Java(TM) SE Runtime Environment (build 1.8.0_73-b02)
Java HotSpot(TM) 64-Bit Server VM (build 25.73-b02, mixed mode)
```
如果你的版本和我不同，看到的源码可能有细微的不同。

# 基础知识
---
本博文的重点是源码分析，关于ThreadPoolExecutor的基础知识出于完整性的需要就一带而过。

ThreadPoolExecutor是Java中形形色色的线程池的基础，它参数最多的一个构造方法如下：
```java
    public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit,
                              BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler)
```
总共七个参数，含义分别如下：
 - corePoolSize:核心池大小，指定了线程池中的线程数量
 - maximumPoolSize:指定了线程池中的最大线程数量
 - keepAliveTime:当线程池线程数量超过corePoolSize时，多余的空闲线程的存活时间
 - unit:keepAliveTime的时间单位
 - workQueue:任务队列，被提交但尚未执行的任务
 - threadFactory:线程工厂，用于创建线程，一般用默认的即可
 - handler:拒绝策略。当任务太多来不及处理，如何拒绝任务
拒绝策略包括以下几种：
 - AbortPoliy:该策略直接抛出异常，阻止系统正常工作
 - CallerRunPolicy:只要线程池未关闭，该策略直接在调用者线程中运行当前被丢弃的任务。任务提交线程的性能极有可能会急剧下降
 - DiscardOldestPolicy:丢弃最老的一个请求，也就是即将被执行的一个任务，并尝试再次提交当前任务
 - DiscardPolicy:默默丢弃无法处理的任务，不予任何处理。如果允许丢失，这是最好的一种方案。

来分析几个常用的线程池的参数加深印象：
### Executors.newSingleThreadExecutor:
```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```
 可见SingleThreadExecutor 其实就是设置核心池大小和最大池大小都为1，此时keepAliveTime已经没有用了，就随意给了个0，最后给了一个无界的等待队列，此时没有指定拒绝策略，默认的拒绝的策略就是AbortPoliy，不过反正队列的是无界的，也不可能遇到拒绝的情况。

### Executors.newCachedThreadPool:
```java
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
```
此时核心池的大小为0，最大池大小非常大，可以认为是无穷。超时时间是60s，因为这里的核心池大小为0，所以所有的线程超过60s没事干都会被回收。等待队列给的是SynchronousQueue，这是一个没有容量的同步队列，所以可以认为所有任务都会被直接提交。从这些参数中可以看出，如果同时有大量任务提交，而这些任务的执行又不那么快的话，系统会开启等量线程来处理，这样的做法会快速耗尽系统资源。
# 核心方法
---
重点分析三个核心方法：
 - execute:提交任务给线程池执行
 - shutdown:关闭线程池，但是线程池不会立即关闭，而是等到等待队列中的任务全部执行完才关闭
 - shutdownNow:立即关闭线程池，将等待队列中的尚未执行的任务返回。

# 线程池的状态
---
成员变量ctl（一个AtomicInteger类型）用于记录线程池当前的一些信息，由线程池当前的状态（runState,如启动，关闭等等）和当前工作线程数（workerCount）组成，分别占用了int的高三位和低29位，线程池状态有以下取值：
```java
    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
```
`<< COUNT_BITS`操作只是为了将数字移动到int的高三位，这四个状态的取值其实就是-1，0，1，2，3。
含义如下（摘自官方文档）：
 - RUNNING:线程池接收新的任务并且处理等待队列中的任务
 - SHUTDOWN:不接收新的任务，但是仍然处理队列中的任务
 - STOP:不接收新的任务，不处理队列中的任务，并且中断正在进行的任务
 - TIDYING:所有的任务都已经结束，工作线程的数量也为0了，将要执行terminated()方法。（注：terminated是一个子类可以通过重写来拓展的方法，即hook）
 - TERMINATED:terminated方法执行结束

从代码上可以看到ctl的初始值是这样的：
```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```
ctlOf是专门用来组装ctl的方法，从这里可以看出初始时线程状态是RUNNING，工作线程数目是0。点开ctlOf方法，发现它其实就是个"或"运算，或运算常被用来进行这种高低位二进制的拼接：
```java
private static int ctlOf(int rs, int wc) { return rs | wc; }
```
在这个类中经常看到的isRunning方法（判断当前线程池状态是否是RUNNING），workerCountOf方法（获取线程池当前工作线程数量），其实就是用"与"操作在ctl上截取需要的信息而已，代码如下：
```java
    private static boolean isRunning(int c) {
        //只有RUNNING状态会比SHUTDOWN小
        return c < SHUTDOWN;
    }
```
```java
    private static int workerCountOf(int c)  { return c & CAPACITY; }
```
CAPACITY其实就是一个用于截取第29位的逻辑尺而已（即低29位全为1，高三位全为0），从它的计算方式可以看出：
```java
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
```

ThreadPoolExecutore的文档上详细说明了线程池状态会发生的转变及其原因，这里我先直接摘抄下来，之后再从源码具体分析：

| 状态转移     |  原因  |
| --------      | :----|
| RUNNING -> SHUTDOWN|   线程池执行了shutdown方法，也可能是在finalize方法中隐式执行的    |
| (RUNNING or SHUTDOWN) -> STOP              |   调用了shutdownNow方法   |
| SHUTDOWN -> TIDYING             |   当队列与线程池都已经为空时    |
|STOP -> TIDYING|线程池为空时|
|TIDYING -> TERMINATED|terminated方法完成|


# 最上层的逻辑
---
ThreadPoolExecutor最表层的逻辑大概是这样的：
![主体逻辑](https://upload-images.jianshu.io/upload_images/10192684-f7bb4db35535dd99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

打开execute方法的代码：
```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        //流程图一
        if (workerCountOf(c) < corePoolSize) {
            //二
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //三
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //五，六，七
        else if (!addWorker(command, false))
            //八
            reject(command);
    }
```

## ①
这个if条件很显然，就不多说了。

## ②
重点看看这个addWorker方法，它会新建线程来执行给定任务：
```java
    /**
     * @param firstTask 
     * @param core     如果为true则将corePoolSize 作为线程池线程数量的边界，否则将maximumPoolSize作为线程池线程数量的边界
     * @return 是否成功新建工作线程（worker）来执行任务
     */
    private boolean addWorker(Runnable firstTask, boolean core) {
        //每当线程池的状态发生变化都会执行retry代码块
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            //一：检测线程池状态
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
            //二：CAS递增workerCount
            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                //如果发现线程池状态发生变化则要重跑整个retry代码块
                if (runStateOf(c) != rs)
                    continue retry;
            }
        }

       //三：新建新的Worker并执行
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

上面的代码虽然看起来长，其实都是CAS之类的细节操作，主题逻辑基本上线性的，注释中已经标出一、二、三步，下面一一讲解：
### 一
看起来比较复杂的一个判断逻辑用于检测线程池当前的状态：
```java
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
```
以下几种情况会直接返回false（也就是没必要启动额外的工作线程执行该任务的情况）：
|情况        | 含义   | 
| --------   | :-----   |
|rs > SHUTDOWN        | 即rs为STOP，TIDYING或者TERMINATED，此时不再接受新的任务|
| rs = SHUTDOWN且firstTask != null        | SHUTDOWN状态下不会再接受新任务除非传进来的firstTask为null，这是为兼顾execute方法中的 `addWorker(null, false)`这种用法，其实传入null是为了启动额外的线程处理队列中剩余的任务     |
| rs = SHUTDOWN且workQueue已经为空     | 承接上一种情况的逻辑，如果连队列都已经为空了，也就没必要启动线程来清理队列中的任务了      |

### 二
一个CAS循环：
```java
            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                //如果发现线程池状态发生变化则要重跑整个retry代码块
                if (runStateOf(c) != rs)
                    continue retry;
            }
```
不断地尝试将workerCount加1（`compareAndIncrementWorkerCount`），如果CAS过程中发现异常（即工作线程数目超过了限制），则立即返回false
```java
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
```
CAPACITY 非常地大，在这里可以认为是无穷，就不考虑了，后面根据core参数决定线程池线程数目的限制是corePoolSize还是maximumPoolSize。

在CAS的过程中，线程池的状态还有可能变化（比如别的线程调用了shutDown方法），所以如果发现rs发生变化，则需要重跑整个外层循环（即retry代码块）：
```java
                c = ctl.get();  // Re-read ctl
                //如果发现线程池状态发生变化则要重跑整个retry代码块
                if (runStateOf(c) != rs)
                    continue retry;
```

### 三
准备工作都做完了，剩下的事情就是启动工作线程来执行任务了，可以看到工作线程对应的是一个叫做Worker的内部类。
```java
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
```
剩下一段代码会有一把可重入锁（ReentrantLock）把他们都锁住，主要原因是workers是一个HashSet类型的变量，它不是线程安全的，它用来存储所有确定要执行的Worker，同时我们还主要到这里维护了一个叫做`largestPoolSize`的变量，用来存储历史最大的线程池大小：
```java
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
```

## ③
```java
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
```
如果线程池的状态还是RUNNING的话，则尝试将任务（command）入队，这里用的是offer方法，offer是阻塞队列的一个非阻塞入队的方法，如果入队成功直接返回true，入队失败则直接返回false。
如果入队成功的话，还要做两件事。
第一件事是再次检查线程池的状态，如果这个时候突然发现状态不是RUNNING，则尝试使用remove方法将该任务从队列中移除，remove方法的内容如下：
```java
    public boolean remove(Runnable task) {
        boolean removed = workQueue.remove(task);
        tryTerminate(); // In case SHUTDOWN and now empty
        return removed;
    }
```
同样是调用了阻塞队列的一个非阻塞方法`remove`，然后用tryTerminate方法尝试终止线程池（因为此时已经不是RUNNING状态了，就可以尝试终止），这里暂时不去细究tryTerminate方法，只要知道这个方法只会在下面两种情况下中止线程池（即将线程池的状态先转换为TIDYING，调用留给子类重写的terminated方法，然后再将状态转换为TERMINATED）：
 - 状态为SHUTDOWN且任务队列为空且工作线程数为0
 - 状态为STOP且工作线程数为0
如果remove也成功的话，此时就直接调用`reject`拒绝该任务了，reject方法的内容如下：
```java
    final void reject(Runnable command) {
        handler.rejectedExecution(command, this);
    }
```
其实就是调用拒绝策略（handler）将任务拒绝。
拒绝策略的类型都是ThreadPoolExecutor的一些内部类，他们都实现了`RejectedExecutionHandler`接口，以`AbortPolicy`为例：

```java
    /**
     * A handler for rejected tasks that throws a
     * {@code RejectedExecutionException}.
     */
    public static class AbortPolicy implements RejectedExecutionHandler {
        /**
         * Creates an {@code AbortPolicy}.
         */
        public AbortPolicy() { }

        /**
         * Always throws RejectedExecutionException.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         * @throws RejectedExecutionException always
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }
```
它对rejectedExecution方法的实现就是直接抛出异常。


第二件事就是发现此时工作线程已经死光光了（`workerCountOf(recheck) == 0`），则调用`addWorker(null, false);`，前面讲addWorker方法时说过第一个参数传null的用法，第一参数传null的目的是为了新建Worker线程清理队列总的剩余任务。


## ⑤⑥⑦⑧
如果加入队列失败的话，此时会再次尝试用`addWorker`提交任务，此时第二个参数传的是false:
```java
        //五，六，七
        else if (!addWorker(command, false))
            //八
            reject(command);
```
这次是以最大线程池大小为限制调用addWorker方法，如果此时线程池中线程的数目没有达到maximumPoolSize，则会再创建一个Worker线程来处理该任务并返回true，否则返回false。
如果addWorker返回false的话，则就会调用之前讲过的reject方法来将任务拒绝掉。

# Worker
---
## AQS的实现
ThreadPoolExecutor的内部类Worker代表一个工作线程，它实现了Runnable接口，继承了AQS：
```java
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
```
这里总算出现了上篇文章讲的AQS。
Worker是对AQS的一个最简单的实现，即一个不可重入锁：
```java
        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
```
这和上一篇文章里我们自己继承AQS实现的SimpleLock是不是很像？
先看看Worker唯一的一个构造方法（在addWorker中经常被调用）：
```java
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
```
发现它先将同步状态（state）设置为-1（现在看来有点奇怪，待会再解释原因），然后将任务设为自己的firstTask字段，这是这个工作线程即将要执行的第一个任务，最后调用之前构造函数传入的ThreadFactory新建线程。
## runWorker
Worker既然代表的是工作线程，那么其核心肯定是run方法，如下：
```java
        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }
```
发现它直接调用了ThreadPoolExecutor中的runWorker方法：
```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```
runWorker的大概逻辑如下图：
![Worker主体逻辑](https://upload-images.jianshu.io/upload_images/10192684-a9a563efe101cb1e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在主循环开始之前有这几行代码：
```java
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
       //用于标记主循环是否是因为异常结束的
        boolean completedAbruptly = true;
```
这里比较让人困惑的地方是，还没有lock怎么就unlock了，
这就让我们回想起了Worker构造方法中将state置为-1，这里是为了将其重新置回0，为什么要多这一步呢？主要是因为从addWorker的代码中我们可以看出从Worker对象新建到开始执行还是需要一段时间的，state为-1就是为了标记这一段时间，为了后面的shutdownNow方法实现做铺垫，具体到后面讲shutdownNow方法时再细说。
主循环的代码如下：
```java
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        }
```
我把整个try代码块都粘了过来，可以看到try代码块中除了主循环，循环外还有一句`completedAbruptly = false;`，可见completedAbruptly为false标记着try代码块是正常结束的，否则是因为抛出异常而结束的。
虽然代码很长，但做的事情其实就是先调用getTask方法将任务从队列中取出来，然后执行，循环往复。
## getTask方法
先具体看一看getTask方法：
```java
    private Runnable getTask() {
        //用于标记是否超时
        boolean timedOut = false; 

        //CAS循环
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            //检查一：线程池状态
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);
            //用于标记线程是否可以超时停止
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            //检查二：工作线程数目以及是否超时
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            //尝试从队列中取任务
            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```
因为getTask一旦返回null，while循环就不得不结束了，所以getTask掌控while循环是否结束的主要逻辑。
在从队列中取任务之前要先做一些检查，检查通过之后才尝试着从队列中取任务返回。
### 检查一
第一轮检查就是代码注释中"检查一"的if语句，在这个检查中，发现线程池的状态出现以下情况则直接返回null让线程结束，可以看到在返回null之前都会先调用`decrementWorkerCount`方法将线程数量减一：

| 情况        | 原因    |
| --------   | :-----   |
| rs > SHUTDOWN        | 此时状态至少时STOP，这种情况先线程池不会再处理队列中的任务      |
| rs = SHUTDOWN 且 队列为空      | 队列中此时已经没有任务，而且线程池的状态已经为SHUTDOWN，也不会再有新任务提交，此时直接返回null结束while循环      |
### 检查二
能够到达"检查二"的，要么此时线程池RUNNING状态，要么是SHUTDOWN状态但是队列中还有任务要执行。
"检查二"的检查条件如下：
```java
(wc > maximumPoolSize || (timed && timedOut)) 
&& 
(wc > 1 || workQueue.isEmpty())
```
重点看`&&`号的左边，`&&`的右边主要是为了防止这么一种情况：当前线程是线程池唯一一个工作线程，但是队列中又有任务要处理的情况，所以不是重点。
`wc > maximumPoolSize`很好理解，如果线程数目已经超过了最大线程数目，那么肯定是需要返回null停止当前线程的。
后面的`(timed && timedOut)`用于判断超时，`timed`用于标记线程是否会受到超时影响，它是这么产生的：
```java
            //用于标记线程是否可以超时停止
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
```
前面在讲构造方法参数含义的时候说过，当线程池中的线程数目超过核心池大小时，会因为超时而停止线程，所以`wc > corePoolSize`很好理解。比较奇怪的是这个`allowCoreThreadTimeOut `这个标记，在构造方法里没有看到，听名字似乎可以让核心池的线程也会因超时而停止，这个标记是从哪里来的呢？其实如果正常使用线程池的话这个标记一般是false，除非从外界调用线程池对象的`allowCoreThreadTimeOut`方法，这个方法的代码如下：
```java
    public void allowCoreThreadTimeOut(boolean value) {
        if (value && keepAliveTime <= 0)
            throw new IllegalArgumentException("Core threads must have nonzero keep alive times");
        if (value != allowCoreThreadTimeOut) {
            //设置allowCoreThreadTimeOut 
            allowCoreThreadTimeOut = value;
            if (value)
                interruptIdleWorkers();
        }
    }
```
另一个超时标记`timeOut`由上一轮循环产生，读一读getTask最下面的几行代码发现，`timeOut`会在上一轮poll超时时被设置为true。
两轮检查都通过之后就可以去队列里去任务，取任务的代码很好理解，就不多说了，如下：
```java
            //尝试从队列中取任务
            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
```

## 执行任务
这部分讲`runWorker`主循环体的代码，如下：
```java
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
```
可以看到这部分代码被worker上了一把锁，至于为什么上锁稍后再说。
在执行任务之前会检查一下线程池的状态，按照源码注释所说，如果至少是STOP状态，则将线程的中断标志置位：
```java
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
```
然后就开始执行任务了，顺序大概是`beforeExcute`->`task.run`->`afterExecute`，beforeExcute和afterExecute其实就是留给子类实现的钩子：
```java
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
```
从最后的finally代码块如下：
```java
                finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
```
`task=null`是为了辅助GC，因为下一轮循环开始处的`getTask`很有可能会阻塞很长时间，而task长时间不释放引用，导致没用的任务长时间不被垃圾回收。
Worker对象内部还维护了一个completedTasks的字段，表示它已经完成的总任务数。

## processWorkerExit
到这个主循环已经结束，Worker线程准备退出了，`processWorkerExit`将完成一些退出前的准备工作，大概的内容我已经写在之前的那种Worker主题逻辑图中了，现在看看代码：
```java
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        //一：从工作者集(workers)中删除该worker
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        //二：尝试停止线程池
        tryTerminate();

        //三：如果符合条件的话则要新建一个Worker来处理
        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }
```
这个方法很重要的是第二个参数，表示线程是否是正常结束的（true表示非正常结束，false表示正常结束），什么样叫正常结束，什么样叫非正常结束呢？我总结如下：
 - 正常结束：通过getTask返回null而结束的while循环
 - 非正常结束：其他各种未知原因程序出现异常

明白了正常情况和非正常情况，前两句代码就很容易明白：
```java
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();
```
如果是非正常结束，则需要在这里将workerCount减1（因为如果是正常结束的话，在getTask返回null之前，getTask就会将其减1），然后代码就开始了之前我在图中画的三步。

### ①
```java
        //从工作者集(workers)中删除该worker
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }
```
之前说过，工作者集（workers）是一个HashSet结构，线程不安全，所以这里要加一把全局的锁。

### ②
```java
        //尝试停止线程池
        tryTerminate();
```
前面虽然说过这个方法，但是没看代码，现在来看看代码：
```java
    final void tryTerminate() {
        for (;;) {
            //第一部分：检查
            int c = ctl.get();
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            if (workerCountOf(c) != 0) { // Eligible to terminate
                //中断一个空闲线程,让它来继续处理SHUTDOWN信号
                //这个方法我稍后讲shutdown方法时会更详细地讲
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            //第二部分：停止线程池
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        terminated();
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```
外面套一层for循环个人认为是因为使用了`compareAndSet`这种CAS操作而做的谨慎考虑。
整个代码其实只有两个部分，一个部分是检查，第二个部分是如果检查通过了话，则要关闭线程池。
只有满足和我之前提到的两种情况才能通过检查。
第二部分线程池关闭的代码则非常简单，就是走的`TIDYING`->`terminated()调用`->`TERMINATED`的流程。可以注意到整个过程被一把mainLock锁住了，所以子类在实现terminated方法时应该是不需要考虑线程安全的。

### ③
尝试停止线程池之后，在某些情况下还需要再建立一个Worker来处理剩下的任务：
```java
        //如果符合条件的话则要新建一个Worker来处理
        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            //新建一个Worker来处理队列中剩余的任务
            addWorker(null, false);
        }
```
判断条件看起来很复杂，但是总结下来就是之前我图中说的三种情况：
| 情况       |原因    | 
| --------   | :----|
| 线程是因为异常结束（completedAbruptly=true）    |  此时线程不是正常原因结束的，所以这个线程本来不应被关闭，所以应该重新创建一个Worker代替它     | 
| 允许核心线程超时（allowCoreThreadTimeOut=true） 且 任务队列不为空   且 worker数量为0        | 此时队列中还有任务没有执行，但是已经没有worker了，所以必须再创建一个Worker      |  
| 不允许核心线程超时（allowCoreThreadTimeOut=false） 且 worker数量小于corePoolSize        | 在不允许核心线程超时的情况下，必须要维持线程数目至少等于corePoolSize，所以当发现线程数目小于 corePoolSize时，必须要新建线程     |  

# shutdown
---
shutdown的逻辑非常简单：
```java
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
             //使用Java安全管理器检查权限
            checkShutdownAccess();
            //将线程池的状态置为SHUTDOWN
            advanceRunState(SHUTDOWN);
            interruptIdleWorkers();
            //由子类实现的hook
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }
```
大多数代码通过注释就能明白，重点看看interruptIdleWorkers方法：
```java
   private void interruptIdleWorkers() {
        interruptIdleWorkers(false);
    }
```
发现它调用了另一个方法：
```java
    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                //onlyOne为true时则只中断一个空闲线程
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
```
这里的算法大概是，遍历工作者集（workers），中断没有在执行任务的Worker。它是怎么判断Worker是否在执行任务呢？这里其实是通过`tryLock`方法能否获得锁来判定的，为什么这个方法能够判定Worker是否在执行任务呢？回想我们之前有一个疑惑一直没解决，就是为什么在runWorker方法的主循环中为什么会用worker把循环体全部锁住，现在答案很清晰了：为了分辨Worker是否在执行任务，如果Worker在执行任务的时候（处在循环体中），该Worker的锁就会处于被占用的状态，tryLock就会返回false，不在执行任务的时候，tryLock就会返回true，所以我们可以用tryLock来判定Worker是否在执行任务（一句话概括答案：保证Worker在执行任务时不会被中断）。


# shutdownNow
---
逻辑与shutdown相似：
```java
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //检查权限
            checkShutdownAccess();
            //将线程池状态设置为STOP
            advanceRunState(STOP);
            interruptWorkers();
            //获取队列中还未执行任务
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
```
它直接就将线程的状态置为STOP了，并且会获取队列中还未执行的任务并返回。
它调用的中断线程的方法是`interruptWorkers`，内容如下：
```java
    private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }
```
发现它其实是调用每个worker对象的interruptIfStarted方法，这个方法的内容如下：
```java
        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```
这个方法顾名思义是只中断已经启动的线程，它是通过`getState() >= 0`来判断的（大于等于0则说明Worker已经启动），为什么这样能判断呢？回忆起我们之前提到Worker构造方法中会先将state置为-1，然后在主循环开始之前通过unlock将state置为0，这么一趟奇怪的操作其实就是为了区分已经启动的Worker和未启动的Worker。

下面再画一张表总结一下shutdown与shutdownNow的区别：

|         | 主体逻辑    |  中断线程使用的方法  |方法效果 |
| --------   | :-----   | :---- |:---- |
| shutdown        | 将runState置为SHUTDOWN，会保证队列中剩余任务的执行      |   interruptWorkers|遍历工作集（workers），中断没有在执行任务的Worker |
| shutdownNow        | 将runState置为STOP，然后返回队列中还没有被执行的任务（这些任务不会再被执行了）      |   worker.interruptIfStarted    | 只要是启动了的Worker，就将其中断|

# ThreadPoolExecutor的线程模型
---
到这里，ThreadPoolExecutor最核心的一些代码已经过了一遍了，可以去总结它的线程模型了，如下图;
![线程模型](https://upload-images.jianshu.io/upload_images/10192684-fc47d7fc67ea962c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


