---
layout: post
title: 图解java.util.concurrent源码（五） CountDownLatch
date: 2018-12-23
categories: java
tags: java 并发
cover: /assets/img/lock.jpg
---

# 引言
---

今天分享一个比较简短一些的源码，那就是concurrent包中我们经常使用的CountDownLatch同步器，"latch"在英文中也是锁的意思，翻译成中文就是“倒数锁”，当你调用了这个类型对象中的await方法后，必须要等待这个锁倒数到0，才能继续运行。

这个类的源码非常短，因为其实它就是对AQS共享模式的一个简单实现而已，如果你还不理解AQS的话，可以去看看我这个系列的[第一篇文章](https://www.dqyuan.top/2018/09/06/abstractqueuesynchronizer.html)和[第三篇文章](https://www.dqyuan.top/2018/09/24/reentrantlock-semaphore.html)，他们详细介绍了AQS的使用与原理，以及AQS经典的两个实现类ReentrantLock与Semophore。

# Demo
---

虽然这篇文章重点讲的是实现，但是出于完整性，这里也给个demo示意其使用，如果你已经知道这个类的使用方法的话，直接跳过这一部分即可。

假设现在有一个大小为15的int数组（全部为0），我想用三个线程将数组中的内容全部置位1（每个线程只需要将5个数组元素置位1即可），使用CountDownLatch即可实现这个过程的同步：

```java
package org.du.test.blogdemo;

import java.util.Arrays;
import java.util.concurrent.CountDownLatch;

public class CountDownLatchDemo {

    private static class Fill extends Thread{

        private int start;

        private int end;

        private int[] array;

        private CountDownLatch countDownLatch;

        public Fill(int start, int end, int[] array, CountDownLatch countDownLatch) {
            this.start = start;
            this.end = end;
            this.array = array;
            this.countDownLatch = countDownLatch;
        }

        @Override
        public void run() {
            for ( int i = start; i < end; i++ ){
                array[i] = 1;
            }
            //该部分任务完成,倒数一个
            countDownLatch.countDown();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        //任务只有3个部分,所以倒数三下即可,构造函数中的3代表需要倒数的次数
        CountDownLatch countDownLatch = new CountDownLatch(3);

        int[] array = new int[15];

        new Fill(0, 5, array, countDownLatch).start();
        new Fill(5, 10, array, countDownLatch).start();
        new Fill(10, 15, array, countDownLatch).start();

        //等待倒数完成
        countDownLatch.await();

        System.out.printf(Arrays.toString(array));
    }

}

```

程序最后输出：

```c-like
[1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
```

所有的元素都被置1了，说明线程同步是正确的。

需要注意的一点是CountDownLatch的计数是无法被重置的，如果你的场景总是需要重置计数的话，最好考虑使用CyclicBarrier(循环栅栏)，我的下一篇文章将会详细分析CyclicBarrier。


# 实现分析
---

从上面的demo可以看出，CountDownLatch的核心方法其实就只有两个：

 - countDown: 用于计数器倒数一次
 - await: 等待计数器倒数完成

点开这两个方法看一下：

```java
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```

```java
    public void countDown() {
        sync.releaseShared(1);
    }
```

发现有点类似与之前讲过的[Semaphore](https://www.dqyuan.top/2018/09/24/reentrantlock-semaphore.html)，因为他们都是AQS在共享模式下的实现，套路是一样的。

Sync内部类实现了AQS接口：

```java
    private static final class Sync extends AbstractQueuedSynchronizer {
                                              ...
```

而当你调用`CountDownLatch(int count)`构造方法时会实例化一个sync实例：

```java
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        //实例化sync同步器
        this.sync = new Sync(count);
    }
```

继续查看Sync的构造器的代码:

```java
        Sync(int count) {
            //这里AQS的state代表计数器的值
            setState(count);
        }
```

回忆一下AQS的相关知识，AQS允许子类通过state变量进行同步，在ReentrantLock的实现中，state同步变量被用作记录线程的冲入次数，而在Semaphore的实现中，state同步变量被用作记录剩余许可数，这里可以看出：

 - **CountDownLatch中的AQS同步状态被用作记录剩余倒数次数**

## countDown的实现

从上面可以看到countDown方法本质上调用的是AQS的`releaseShared`，回忆一下AQS的知识，`releaseShared`本质上是调用子类实现的`tryReleaseShared`类尝试释放锁的，Sync的实现如下：

```java
        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            //使用CAS循环更新  计数器 减一
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
```

可以看出就是一个CAS循环，不把同步状态state减1誓不摆休。其中`compareAndSetState`是由父类AQS提供的用于操作同步状态state的方法。

## await的实现

await调用的其实就是AQS的`acquireSharedInterruptibly`方法，而`acquireSharedInterruptibly`本质上是调用子类实现的`tryAcquireShared`方法类不断尝试获得锁的，Sync中的实现如下：

```java
        protected int tryAcquireShared(int acquires) {
            //返回1会不停地传播下去
            return (getState() == 0) ? 1 : -1;
        }
```

代码只有一行，但是要看懂的话需要了解一些AQS共享模式的知识，如果子类实现的`tryAcquireShared`返回一个大于等于0的数字的话，线程获得锁，除了会将自己的节点设置为头结点外，还会继续唤醒后继的一个节点（代码中称之为“传播”, PROPAGATE）;如果返回0的话，则线程获得锁，然后就像独占模式中一样将自己的节点设置为头节点；返回一个小于0的数字的话，则线程无法获得锁，等待它的就将是被阻塞。

这里画了张表格总结了一下：

| `tryAcquireShared`返回值        | 导致的行为    |
| :--------   | :-----   |
| >0        | 线程获得锁，将自己的节点设置为头结点，继续唤醒后继的一个节点      |  
| =0        | 线程获得锁，将自己的节点设置为头结点      |  
| <0        | 线程无法获得锁      |  


这样我们就能看懂这个返回值了，如果`getState`发现同步状态已经倒数到0了，则一直返回1，这样就能够从头节点不停地传播下去，直到唤醒所有正在`await`的线程。

如果发现还没有倒数到0，则始终返回-1，这样所有正在`await`的线程就会一直阻塞下去。

# End
---

这一篇分享比较短，有划水的嫌疑，其实我是在为下一篇介绍`CyclicBarrier`的文章作铺垫。

![滑稽.jpg](https://upload-images.jianshu.io/upload_images/10192684-626047d51fc749d2.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

