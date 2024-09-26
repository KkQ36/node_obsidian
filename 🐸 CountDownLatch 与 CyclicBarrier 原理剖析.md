# CountDownLatch 原理探究
## 案例介绍
在日常开发中经常会遇到需要在主线程中开启多个线程去并行执行任务 ， 并且主线程需要等待所有子线程执行完毕后再进行汇总的场景。
在 `CountDownLatch` 出现之前一般都使用线程的 `join()` 方法来实现这一点，但是 `join()` 方法不够灵活 ，不能够满足不同场景的需要。
所以 JDK 开发组提供了 `CountDownLatch` 这个类，使用它可以优雅的做到线程同步：
```java
public class Main {  
    private static final CountDownLatch countDownLatch = new CountDownLatch(2);  
    private static final AtomicInteger integer = new AtomicInteger(0);  
  
    public static void main(String[] args) throws InterruptedException {  
        Thread thread1 = new Thread(() -> {  
            try {  
                Thread.sleep(1000);  
            } catch (InterruptedException e) {  
                System.out.println("thread1 sleep interrupted");  
            } finally {  
                countDownLatch.countDown();  
            }  
            integer.addAndGet(1);  
            System.out.println("thread1 end");  
        });  
  
        Thread thread2 = new Thread(() -> {  
            try {  
                Thread.sleep(1000);  
            } catch (InterruptedException e) {  
                System.out.println("thread2 sleep interrupted");  
            } finally {  
                countDownLatch.countDown();  
            }  
            integer.addAndGet(1);  
            System.out.println("thread2 end");  
        });  
		// 启动线程
        thread1.start();  
        thread2.start();  
        System.out.println("wait all thread over");  
        
		// 等待线程执行完成
        countDownLatch.await();  
        System.out.println("all thread over，integer = " + integer.get());  
    }  
  
}
```
输出结果如下：
```
wait all thread over
thread1 end
thread2 end
all thread over，integer = 2
```
在如上代码中，创建了一个 `CountDownLatch` 实例，因为主线程有两个子线程所以构造函数的传参为 2 。
主线程调用 `countDownLatch.await()` 方法后会被阻塞 。
子线程执行完毕后调用 `countDownLatch.countDown()` 方法让 `countDownLatch` 内部的计数器减 l ，所有子线程执行完毕并调用 `countDown()` 方法后 计数器会变 为 0 ，这时候主线程 的 `await()` 方法才会返回 。
其实上面的代码还不够优雅 ，在项目实践中一般都避免直接操作线程，而是使用线程池来管理。
使用线程池时传递的 参数是 Runable 或 者 Callable 对象 ，这时候你没有办法直接调用这些线程的`join()`方法，这就需要选择使用`CountDownLatch` 了
```java
public class Example_ThreadPool {  
  
    private static final CountDownLatch countDownLatch = new CountDownLatch(2);  
  
    public static void main(String[] args) throws InterruptedException {  
        ExecutorService executorService = Executors.newFixedThreadPool(2);  
        executorService.execute(()->{  
            System.out.println("线程 1 执行");  
            countDownLatch.countDown();  
        });  
        executorService.execute(()->{  
            System.out.println("线程 2 执行");  
            countDownLatch.countDown();  
        });  
        countDownLatch.await();  
        System.out.println("all thread over");  
        executorService.shutdown();  
    }  
  
}
```
输出结果如下：
```
线程 1 执行
线程 2 执行
all thread over
```
## 实现原理探究
### 构造方法
从 CountDownLatch 的名字就可以猜测其内部应该有个计数器，并且这个计数器是递减的，来看一下它的类图结构：
![[Pasted image 20240923201510.png|600]]
从类图可以看出，`CountDownLatch` 类是使用 AQS 实现的，其静态内部类 Sync 通过继承 AQS 来实现类似锁的结构（和 ReentrantLock 结构很相似），来看一下它的构造函数：
```java
public CountDownLatch(int count) {  
    if (count < 0) throw new IllegalArgumentException("count < 0");  
    this.sync = new Sync(count);  
}

Sync(int count) {  
    setState(count);  
}
```
其实就是将 AQS 的 `state` 设置成了传入的 `count`；看到这里，它的实现逻辑就不难猜测了，我们来看几个关键的方法。
### void await() 方法
当线程调用 `CountDownLatch` 的 `await` 方法的时候，当前线程会被阻塞直到下面两种情况的时候会停止等待。
- 其他线程调用完了 `CountDownLatch` 的 `countDown` 方法计数器变为 0 的时候。
- 其他线程调用了当前线程的 `interrupt()` 方法导致线程被中断。
```java
public void await() throws InterruptedException {  
    sync.acquireSharedInterruptibly(1);  
}

public final void acquireSharedInterruptibly(int arg)  
    throws InterruptedException {  
    if (Thread.interrupted() ||  
        (tryAcquireShared(arg) < 0 &&  
         acquire(null, arg, true, true, false, 0L) < 0))  
        throw new InterruptedException();  
}
```
在 Sync 类中，`tryAcquireShared()` 方法的实现逻辑是这样的：
```java
protected int tryAcquireShared(int acquires) {  
    return (getState() == 0) ? 1 : -1;  
}
```
当线程调用这个方法的时候，首先会检查线程是否被中断，然后调用 Sync 中实现的方法去尝试获取共享资源，如果获取失败，也就是当前的计数器不为 0 的时候，会调用 `acquire()` 方法使得线程最终被阻塞挂起。
方法中最终通过 `LockSupport` 来挂起线程，使用这种方法挂起的线程被调用中断方法的时候，不会立刻抛出异常，而是会设置标志位。
当然在 AQS 中，会自动帮助我们抛出异常，所以底层使用什么对我们来说都是相同的；
但是在线程池中也是通过这样的方式挂起线程，当停止的时候，工作线程也不会立刻抛出异常，我们可以对这个标志位进行判断，然后做一些操作。
```java
LockSupport.park(this);
```
### void countDown() 方法
线程调用该方法后，计数器的值递减 ，递减后如果计数器值为 0 则唤醒所有因调用 `await()` 方法而被阻塞的线程，否则什么都不做。下面看下 `countDown()` 方法是如何调用 AQS 的方法的。
```java
public void countDown() {  
    sync.releaseShared(1);  
}

// AQS 类
public final boolean releaseShared(int arg) {  
    if (tryReleaseShared(arg)) {  
        signalNext(head);  
        return true;  
    }  
    return false;  
}

// Sync
protected boolean tryReleaseShared(int releases) {  
        // Decrement count; signal when transition to zero  
        for (;;) {  
            int c = getState();  
            // 当前状态值为 0，直接返回
            if (c == 0)  
                return false;  
			// 调用 CAS 让计数器减一
            int nextc = c - 1;  
            if (compareAndSetState(c, nextc))  
                return nextc == 0;  
        }  
    }  
}
```
最终会调用到 Sync 类中实现的 `tryReleaseShared` 方法，这个方法检测当前状态值，如果为 0 直接返回（属于异常状态，不做任何处理）。
否则使用 CAS 操作使计数器减一，如果在本次中使得计数器变为 0，返回到 AQS 的 `releaseShared` 方法，唤醒队列中所有等待的线程。
```java
if (acquired) {  
    if (first) {  
        node.prev = null;  
        head = node;  
        pred.next = null;  
        node.waiter = null; 
        // 如果是 share 模式，当前线程苏醒后，自动唤醒下一个线程，下一个线程会循环等待直到获取锁，
        // 然后继续唤醒其他线程，最终所有线程都将会被唤醒。 
        if (shared)  
            signalNextIfShared(node);  
        if (interrupted)  
            current.interrupt();  
    }  
    return 1;  
}
```

# CyclicBarrier 回环屏障原理探究
## 基本介绍
上节介绍的 CountDownLatch 在解决多个线程同步方面相对于调用线程的 join 方法己经有了不少优化，但是 CountDownLatch 的计数器是**一次性的**，也就是等到计数器值变为0 后，再调用 CountDownLatch 的 await 和 countdown 方法都会立刻返回，这就起不到线程同步的效果了 。
所以为了满足计数器可以重置的需要， JDK 开发组提供了 CyclicBarrier类 ， 并且 CyclicBarrier 类的功能 并不限于 CountDownLatch 的功能 。
从字面意思理解，CyclicBarrier 是回环屏障的意思 ，它可以让一组线程全部达到一个状态后再全部同时执行 。
这里之所以叫作回环是因为当所有等待线程执行完毕，并重置 CyclicBarrier 的状态后它可以被重用。之所以叫作屏障是因为线程调用 await 方法后就会被阻塞，这个阻塞点就称为屏障点，等所有线程都调用了 await 方法后，线程们就会冲破屏障，继续向下运行。
比如，我们来看一个例子：
![[Pasted image 20240924212023.png|800]]
有两个线程，当它们同时到达屏障的时候，count 变为零，线程得以穿过屏障；我们可以简单理解为两个线程需要拼车，当车上的人数达到 count 的时候，才能发车继续向下执行；同时发车后，又会有一辆新的车继续提供给线程乘坐。
## 案例介绍
