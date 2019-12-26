# java并发源码：Condition

### 比对Object的监视器方法和Condition接口

| 对比项                        | Object | Condition                                |
| :------------------------- | ------ | ---------------------------------------- |
| 前置条件                       | 获取对象的锁 | 调用Lock.lock()获取锁                                                                                                     调用Lock.newCondition()获取condition对象。 |
| 调用方式                       | 直接调用   | 直接调用                                     |
| 等待队列个数                     | 1个     | 多个                                       |
| 当前线程释放锁并进入等待状态             | 支持     | 支持                                       |
| 当前线程释放锁并进入等待状态，在等待状态中不响应中断 | 不支持    | 支持                                       |
| 当前线程释放锁并进入超时等待状态           | 支持     | 支持                                       |
| 当前线程释放锁并进入等待状态到将来的某个时间     | 不支持    | 支持                                       |
| 唤醒等待队列中的一个线程               | 支持     | 支持                                       |
| 唤醒等待队列中的全部线程               | 支持     | 支持                                       |

Condition对象是依赖Lock对象的。

### API方法

| 方法名称                               | 描述                                       |
| ---------------------------------- | ---------------------------------------- |
| void await()                       | 当前线程进入等待状态知道被通知或中断，当前线程将进入运行状态且从await()方法返回的情况：                                                                                    1其它线程中断当前线程                                                                                       2如果当前等待线程从await()返回，则表明该线程已经获取了Condition对象所对应的锁。 |
| void awaituninterruptibly()        | 当前线程进入等待状态直到被通知，从方法名称上可以看出该方法对中断不敏感。     |
| long awaitNanos(long nanosTimeout) | 当前线程进入等待状态直到被通知，中断或者超时。返回值表示剩余的时间，如果在nanosTimeout纳秒之前被唤醒，那么返回值就是(nanosTimeout-实际耗时)。如果返回值是0或者负数，那么可以认定已经超时。 |
| boolean awaitUntil(Date deadli)    | 当前线程进入等待状态直到被通知，中断或者直到某个时间，如果没有到指定时间就被通知，方法返回true，否则到了指定时间，方法返回false |
| void signal()                      | 唤醒一个等待在Condition的线程，该线程从等待方法返回前必须获得与Condition相关联的锁。 |
| void signalAll()                   | 唤醒所有等待在Condition的线程，能够从等待方法返回的线程必须获得与Condition相关联的锁。 |

###  实现分析

#### 等待队列：

​	等待队列是一个FIFO的队列，每个节点都包含了一个线程引用，该线程就是在Condition对象等待的线程，如果一个线程调用了Condition.await()方法，那么该线程将会释放锁，构成节点并加入等待队列并进入等待状态。

​	一个对象拥有一个同步队列和等待队列。

​	Lock（同步器）拥有一个同步队列和多个等待队列。

#### 等待

ConditionObject的await方法。

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
  	//当前线程加入等待队列
    Node node = addConditionWaiter();
  	//释放同步状态，也就是释放锁。
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

调用await()方法，相当于从同步队列的首节点（获取了锁的节点）移动到Condition的等待队列中。

#### 通知

```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

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

```java
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

​	当前线程必须是获取了锁的线程，接着获取等待队列的首节点，将其移动到同步队列并使用LockSupport唤醒节点中的线程。