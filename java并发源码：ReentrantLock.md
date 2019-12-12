# java并发源码：ReentrantLock

## 结构

 ![YTZ61BCKT}LAI51~NM%O2QM](C:\Users\Admin\Desktop\images\YTZ61BCKT}LAI51~NM%O2QM.png)

可见ReentrantLock实现了Lock接口。

### lock

**lock接口定义了五个方法。**

`void lock();`：获取锁。

`void lockInterruptibly() throws InterruptedException;`：可中断的获取锁，该方法会响应中断。

`boolean tryLock();`：尝试非阻塞的获取锁。

`boolean tryLock(long time, TimeUnit unit) throws InterruptedException;`：超时的获取锁。

`void unlock();`：释放锁。

`Condition newCondition();`：获取等待通知组件。该组件与当前的锁绑定，当前线程只有获得锁，才能调用该组件的wait()方法，调用后，当前线程释放锁。



​	重入锁ReetrantLock，表示该锁能够支持一个线程对资源的重复加锁。该锁还支持获取锁时的公平和非公平性选择。公平锁是指等待时间最长，也就是保证先来的线程最优先获得锁，锁获取是顺序的。实现公平锁需要维护一个有序队列，性能较低。非公平锁随机获取锁，会出现有些线程一直无法获取锁，出现饥饿现象。

### 实现可重入

​	重进入是指任何线程在获取锁会后能够再次获取该锁而不会被锁所阻塞。

-  线程的再次获取锁。
- 锁的最终释放。线程重复N次获取了锁，随后在第N次释放该锁后，其它线程能够获取到该锁。

### 方法解读

##### 非公平锁：

nonfairTryAcquire();

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
	//aqs中定义的state变量
    int c = getState();
    if (c == 0) {
      	//CAS方式改变state值。
        if (compareAndSetState(0, acquires)) {
          	//将当前线程保留，方便下次直接判断。
            setExclusiveOwnerThread(current);
            return true;
        }
    }
  	//如果当前线程是获取锁的线程，同步状态值+1，并返回true
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

tryRelease()

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
  	//如果当前线程不是获取锁的线程，抛出异常。
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
  	//只有当c等于0时，表示同步状态完全释放，才能返回true。
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

##### 公平锁：

tryAcquire()

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
      	//与非公平锁对比，
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

```java
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
  	//同步队列中当前节点是否与前驱节点的判断。如果返回true，则表示有其它线程比当前线程更早的请求获取锁，因此需要等待前驱线程获取并释放锁之后才能继续获取锁。
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

