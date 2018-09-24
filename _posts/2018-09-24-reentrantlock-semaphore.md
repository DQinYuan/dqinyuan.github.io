---
layout: post
title: 图解java.util.concurrent源码（三） Reentrantlock && Semaphore
date: 2018-09-24
categories: java
tags: 并发 java
---

# 引言
Reentrantlock和Semaphore分别是AQS在独占模式和共享模式下经典的实现，在理解AQS的情况下看这两个类的代码会感到非常简单，如果还没理解AQS的话，建议先读我这个系列的[第一篇文章](http://www.dqyuan.top/2018/09/06/abstractqueuesynchronizer.html)

# 复习AQS

回忆一下AQS，AQS中维护了一个state同步状态，它的子类只需要实现以下几个方法，并在方法中修改判断state的值即可：

**独占模式**的同步器（比如Reentrantlock）需要实现：

- tryAcquire
- tryRelease

**共享模式**的同步器（比如Semaphore）则需要实现：

- tryAcquireShared
- tryReleaseShared



如果想要进一步使用AQS的ConditionObject进行线程间同步的话，则子类还应该实现下面的方法：

- isHeldExclusively

下面就中点分析这几个方法。



# Reentrantlock



打开ReentrantLock最常用的三个方法看看（分别是lock,unlock和newCondition），果然全部委托给了叫做sync的内部类对象：

```java
	public void lock() {
		sync.lock();
	}
```

```java
	public void unlock() {
		sync.release(1);
	}
```

```java
	public Condition newCondition() {
		return sync.newCondition();
	}
```

而Sync内部类其实就是AQS的实现：

```java
	abstract static class Sync extends AbstractQueuedSynchronizer
```

而Reentrantlock中还有两个Sync的子类内部类：

```java
    static final class NonfairSync extends Sync 
```

```java
	static final class FairSync extends Sync
```



在ReentrantLock中真正使用的是这两个子类，分别对应非公平锁与公平锁。公平锁能够保证线程按照先进先出（FIFO）的方式获得锁，但是一般认为公平锁的性能不如非公平锁。

下面我们带着两个问题继续阅读：

- ReentrantLock是如何实现可重入（即同一个线程可以持有该锁的情况下多次lock）的？
- 公平锁的FIFO是怎么实现的呢？



### 非公平锁

非公平锁NonfairSync的lock方法实现：

```java
		final void lock() {
			if (compareAndSetState(0, 1))
				setExclusiveOwnerThread(Thread.currentThread());
			else
				acquire(1);
		}
```

发现它会先尝试一下立即获得锁，如果失败的话则退化为正常AQS获锁流程（即父类AQS中的acquire方法），这里注意到acquire方法接收的参数是1。

tryAcquire方法的实现：

```java
		protected final boolean tryAcquire(int acquires) {
			return nonfairTryAcquire(acquires);
		}
```

发现它直接调用的是父类的nonfairTryAcquire方法：

```java
		final boolean nonfairTryAcquire(int acquires) {
			final Thread current = Thread.currentThread();
			int c = getState();
			//state == 0表示该锁处于空闲状态
			if (c == 0) {
			    //获得锁成功
				if (compareAndSetState(0, acquires)) {
					setExclusiveOwnerThread(current);
					return true;
				}
			} else if (current == getExclusiveOwnerThread()) {   //线程重入的情况
				int nextc = c + acquires;
				if (nextc < 0) // overflow
					throw new Error("Maximum lock count exceeded");
				setState(nextc);
				return true;
			}
			return false;
		}
```

如果发现锁处于空闲状态（`state == 0`），则尝试获得锁，否则的话，先判断一下重入的情况，如果是重入的情况（`current == getExclusiveOwnerThread()`），则将同步状态state加1（`int nextc = c + acquires;`），这里的`acquires`的值只可能是1，因为我们之前看到`lock`方法中始终调用的是`acquire(1)`。

再看一下tryRelease方法，tryRelease方法在父类Sync中，也就是说公平锁与非公锁共用的是同一个tryRelease方法：

```java
		protected final boolean tryRelease(int releases) {
			int c = getState() - releases;//减1
            //如果释放的线程不是持有锁的线程，则抛出异常
			if (Thread.currentThread() != getExclusiveOwnerThread())
				throw new IllegalMonitorStateException();
			boolean free = false;
			if (c == 0) {   //锁已经释放完全的状态
				free = true;
				setExclusiveOwnerThread(null);
			}
			setState(c);
			return free;
		}
```



大体上做的事情就是将同步状态state减1，如果发现减到了零的话，则通过`setExclusiveOwnerThread`将AQS的`exclusiveOwnerThread`变量置空，如果已经减到零了，线程再次调用unlock方法的话，则会因为`Thread.currentThread() != getExclusiveOwnerThread()`的判断条件抛出`IllegalMonitorStateException`异常。



看到这里我们可以回答上面提出的第一个问题了：



**ReentrantLock是如何实现可重入的呢？**

答：通过维护同步状态state的含义为“线程重入的次数”，每次线程重入将其加1，释放锁将其减1，直到减成0，将其彻底释放。



顺手去看看为了支持线程间同步（`newCondition`）而实现的`isHeldExclusively`方法（位于Sync类中）：

```java
		protected final boolean isHeldExclusively() {
			return getExclusiveOwnerThread() == Thread.currentThread();
		}
```

发现非常简单，就是判断一下持有锁的线程是否是当前线程，我在第一篇将AQS的文章中说过，ConditionObject的signal方法会首先调用一下`isHeldExclusively`方法确认调用方法的线程是否持有锁。



### 公平锁

FairSync的lock方法的实现：

```java
		final void lock() {
             //非公平锁与公平锁的不同之处一
			acquire(1);
		}
```



发现非常简单粗暴，直接调用AQS父类的`acquire`方法，AQS中维护的CLH队列就是FIFO的，所以这里直接调用acquire即可。而之前的非公平锁的“非公平”又体现再哪里呢？重看一下NonfairSync的lock方法，发现其实就体现在：线程会先尝试一次“插队”，直接设置state获得锁，然后才会调用acquire方法走FIFO的CLH队列，在这个过程中有可能造成CLH队列中等待的线程被后来的线程给“插队”了，就是这个"插队"的行为导致了“不公平”。

上述的修改依旧没能完全制止线程插队的机会，AQS的acquire方法中也会先尝试先用`tryAcquire`方法插队，然后才进入CLH队列，所以FairSync对`tryAcquire`方法也进行了细微的修改（相比NonfairSync）：

```java
		protected final boolean tryAcquire(int acquires) {
			final Thread current = Thread.currentThread();
			int c = getState();
			if (c == 0) {// 初始化状态
				//hasQueuedPredecessors()是公平锁与非公平锁的区别二
				//这个方法来自于AQS
				//hasQueuedPredecessors判断当前线程是否是CLH队列的队头
				//如果在CLH队列中没有前继且CAS成功才能成功获得锁
				if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
					setExclusiveOwnerThread(current);
					return true;
				}
			} else if (current == getExclusiveOwnerThread()) {// 重入
				int nextc = c + acquires;
				if (nextc < 0)
					throw new Error("Maximum lock count exceeded");
				setState(nextc);
				return true;
			}
			return false;
		}
	}
```



可以看出这段代码和NonfairSync的`tryAcquire`基本相同，除了在获得锁的判断条件上添加了一个`hasQueuedPredecessors`，这个方法来自于父类AQS，如果当前线程是CLH队列的队头则返回false，否则返回true。

为什么要做这一层防护呢？因为在AQS的acquire方法中，线程仍然会先尝试调用`tryAcquire`方法插个队，之后才进入`acquireQueued`方法：

```java
    //AQS中的acquire方法
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

线程在进入`acquireQueued`方法之后就彻底是FIFO的了，所以要在前面的`tryAcquire`再进行一道防护，防止在这里"插队"。



上面的这段文字就回答了之前提出的第二个问题，"**公平锁的FIFO是怎么实现的呢？**"



`hasQueuedPredecessors`方法在将AQS时候漏讲了，这里补充一下：

```java
    public final boolean hasQueuedPredecessors() {
        Node t = tail; 
        Node h = head;
        Node s;  //s代表等待队列的第一个节点
        //h != t 是为了判断CLH队列为空的情况
        //(s = h.next) == null 说明此时有另一个线程正在尝试成为头节点，详见AQS的acquireQueued方法
        //s.thread != Thread.currentThread()  此线程不是等待的头节点
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```

判断条件还是有点难理解的，`h != t`比较显然，是为了判断CLH队列为空的情况，接下来的条件是在队列不为空的情况下进行判断，我逐个分析：

- (s = h.next) == null

  首先复习一下AQS中的CLH队列，它的头结点代表当前获得锁的线程，而头节点的下一个节点才代表等待队列的第一个线程。

  所以这里先通过`s = h.next`取到等待队列的第一个节点赋给s。

  这里`h.next`有可能为null，这就要复习一下AQS的acquireQueued方法了，当等待队列的第一个线程获得锁时，它会将头节点的next置空，这个置空next的线程显然是调用`hasQueuedPredecessors`的前继之一，所以返回true

- s.thread != Thread.currentThread()

   当明白s节点代表的就是等待队列的第一个的时候，这个也就很简单了，如果第一个不是当前线程，则肯定是存在前继的，返回true即可。



# Semaphore

Semaphore用来在并发下管理数量有限的资源，是典型的共享模式下的AQS的实现。

和ReentrantLock一样，也分为公平模式和非公平模式。

Semaphore的关键方法如下：

- acquire  获得许可
- release  释放许可

Semaphore并不支持使用CondionObject进行线程间的同步。

看看acquire方法：

```java
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```

```java
    public void acquire(int permits) throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireSharedInterruptibly(permits);
    }
```

直接调用了AQS的`acquireSharedInterruptibly`方法，表明以共享模式使用AQS



再看看release方法：

```java
    public void release() {
        sync.releaseShared(1);
    }
```

```java
    public void release(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.releaseShared(permits);
    }
```

也是直接调了AQS的`releaseShared`方法，共享模式释放。



### 非公平锁

NonfairSync的`tryAcquire`方法：

```java
        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
```

再去其父类看`nonfairTryAcquireShared`方法：

```java
        final int nonfairTryAcquireShared(int acquires) {
            //CAS循环
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```

很好懂，只要明确一点就能够看懂Semaphore的代码：

- **Semaphore的AQS中的同步状态state代表的是剩余许可的数量**

上面那段代码其实就是通过CAS循环不断尝试减少响应数量的许可。



`tryRelease`方法也非常简单，就是通过CAS循环不断尝试增加相应数量的许可：

```java
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
```



### 公平锁

和ReentrantLock一样，就是加了一个`hasQueuedPredecessors`的判断而已：

```java
        protected int tryAcquireShared(int acquires) {
            for (;;) {
                //和非公锁的区别
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```













