# java并发源码：ReentrantReadWriteLock

## 结构

![https://img-blog.csdnimg.cn/20191223160437939.png]()

可见ReentrantReadWriteLock实现了ReadWriteLock接口。

ReadWriteLock接口定义了两个接口:

`Lock readLock()`

`Lock writeLock()`

​	ReentrantLock是排他锁，在同一时刻只允许一个线程进行访问，而读写锁在同一时刻可以允许多个线程访问。但是在写线程访问时，所有的读线程以及写线程均被堵塞，防止脏读。读写锁维护了一对锁，一个读锁和一个写锁。

## 内部工作状态方法：

| 方法名称                    | 描述                                       |
| ----------------------- | ---------------------------------------- |
| int getReadLockCount()  | 返回当前读锁被获取的次数。该次数不等于获取读锁的线程数。例如：一个线程获取了N次读锁，占据该读锁的线程数是1，返回N。 |
| int getReadHoldCount()  | 返回当前线程获取读锁的次数。                           |
| boolean isWriteLocked() | 判断写锁是否被获取。                               |
| int getWriteHoldCount() | 返回当前写锁被获取的次数。                            |

## 读写状态的设计

​	ReentrantLock同步状态表示锁被一个线程重复获取的次数。而读写锁的自定义同步器需要在同步状态上维护多个读线程和一个写线程的状态。

​	读写锁将变量切分成了两个部分，高16位表示读，低16位表示写。通过位运算来判断读和写的状态。假设当前状态是S，写状态等于S & 0x0000FFFF(将高16位全部抹去)，读状态等于S>>>16。当写状态增加1时，等于S+1，当读状态增加1时，等于S+(1<<16)，也就是S+0x00010000。

## 写锁的获取和释放

```java
protected final boolean tryAcquire(int acquires) {
    /*
     * Walkthrough:
     * 1. If read count nonzero or write count nonzero
     *    and owner is a different thread, fail.
     * 2. If count would saturate, fail. (This can only
     *    happen if count is already nonzero.)
     * 3. Otherwise, this thread is eligible for lock if
     *    it is either a reentrant acquire or
     *    queue policy allows it. If so, update state
     *    and set owner.
     */
    Thread current = Thread.currentThread();
    //获取共享变量state
    int c = getState();
  	//获取写锁数量
    int w = exclusiveCount(c);
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
      	//存在读锁或者当前获取线程不是已经获取写锁的线程
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
       //当前线程持有写锁，为重入锁，+acquires即可
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

​	如果存在读锁，则写锁不能被获取。避免脏读发生。

```java
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}
```

## 读锁的获取与释放

​	读锁是一个支持重进入的共享锁，他能被多个线程同时获取。

```java
protected final int tryAcquireShared(int unused) {
    /*
     * Walkthrough:
     * 1. If write lock held by another thread, fail.
     * 2. Otherwise, this thread is eligible for
     *    lock wrt state, so ask if it should block
     *    because of queue policy. If not, try
     *    to grant by CASing state and updating count.
     *    Note that step does not check for reentrant
     *    acquires, which is postponed to full version
     *    to avoid having to check hold count in
     *    the more typical non-reentrant case.
     * 3. If step 2 fails either because thread
     *    apparently not eligible or CAS fails or count
     *    saturated, chain to version with full retry loop.
     */
    Thread current = Thread.currentThread();
    int c = getState();
  	//写锁不等于0的情况下，验证是否是当前写锁尝试获取读锁
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
  	//获取读锁数量
    int r = sharedCount(c);
 	 //CAS操作尝试设置获取读锁 也就是高位加1
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
      	//当前线程第一个并且第一次获取读锁，
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
          //当前线程是第一次获取读锁的线程
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
           // 当前线程不是第一个获取读锁的线程，放入线程本地变量
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}
```

​	如果有线程获取了写锁，则当前线程获取读锁失败，进入等待状态。如果当前线程获取了写锁或者写锁未被获取，则当前线程增加读状态， 成功获取读锁。



```java
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count;
    }
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
```

​	读锁的每次释放均减少读状态，减少的值为(1<<16)

## 锁降级

​	锁降级是指写锁降级为读锁。如果当前线程拥有写锁，然后将其释放，最后再获取读锁，这种分段完成的过程不能称之为锁降级。锁降级是指：拥有写锁，再获取读锁，随后释放写锁的过程。

​	锁降级中读锁的获取是否必要：主要是为了保证数据的可见性，如果当前线程不获取读锁而是直接释放写锁，假设此刻另一个线程（T）获取了写锁，并修改了数据，那么当前线程则无法感知到线程T的数据更新。如果当前线程获取读锁，则遵循锁降级的步骤，则线程T会被阻塞。直到当前线程使用数据并释放 锁之后，线程T才能获取写锁进行数据更新。