## java并发源码：Synchronized

## synchronized加锁方式：

- 修饰实例方法：锁是当前对象。
- 修饰静态方法：锁是当前类的class对象。
- 修饰代码块：锁是synchronized括号里配置的对象。

​	JVM基于进入和退出Monitor对象来实现方法同步和代码块同步的。代码块同步是用monitorenter 和monitorexit指令实现的。

```java
 public void test() {
   synchronized (this) {
   x++;
   }
 }
```

>     Code:
>        0: aload_0
>        1: dup
>        2: astore_1
>        3: monitorenter
>        4: aload_0
>        5: dup
>        6: getfield      #2                  // Field x:I
>        9: iconst_1
>       10: iadd
>       11: putfield      #2                  // Field x:I
>       14: aload_1
>       15: monitorexit
>       16: goto          24
>       19: astore_2
>       20: aload_1
>       21: monitorexit
>       22: aload_2
>       23: athrow
>       24: return
>     Exception table:
>        from    to  target type
>            4    16    19   any
>           19    22    19   any
>

可以看到3的monitorenter指令和15的monitorexit指令

synchronized用的锁是存在java对象头里的。32位虚拟机java对象头的存储结构：

| 锁状态  | 25bit       | 4bit   | 1bit是否偏向锁 | 2bit锁标志位 |
| ---- | ----------- | ------ | --------- | -------- |
| 无锁状态 | 对象的hashcode | 对象分代年龄 | 0         | 01       |

在运行期间，MarkWord数据会发生动态变化

| 锁状态  | 25bit |       | 4bit   | 1bit   | 2bit |
| ---- | ----- | ----- | ------ | ------ | ---- |
|      | 23bit | 2bit  |        | 是否是偏向锁 | 锁标志位 |
| 轻量级锁 |       |       |        |        | 00   |
| 重量级锁 |       |       |        |        | 10   |
| GC标记 |       |       |        |        |      |
|      | 线程ID  | Epoch | 对象分代年龄 | 1      | 01   |

在64位虚拟机下，MarkWord存储结构：

|  锁状态 | 25bit  | 31bit    | 1bit     | 4bit | 1bit | 2bit |
| ---: | ------ | -------- | -------- | ---- | ---- | ---- |
|      |        |          | cms_free | 分代年龄 | 偏向锁  | 锁标志位 |
|   无锁 | unused | hashCode |          |      | 0    | 01   |
|  偏向锁 |        |          |          |      | 1    | 01   |

## synchronized优化

jdk1.6为了减少获得锁和释放锁带来的性能消耗，对synchronized获取锁进行了优化。

无锁->偏向锁->轻量级锁->重量级锁。

### 1、偏向锁

​	大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。当一个线程访问同步块并获取锁时，会在对象头的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要进行CAS操作来加锁和解锁，只需简单地测试一下对象头的MarkWord里是否存储着指向当前线程的偏向锁。如果测试成功，表示线程已经获得了锁。如果测试失败，则需要再测试一下MarkWord中偏向锁的标识是否设置成1（表示当前是偏向锁）：如果没有设置，则使用CAS竞争锁；如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。



![偏向锁](https://user-gold-cdn.xitu.io/2019/10/8/16da8c14ac71acf8?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 2、轻量级锁

### 轻量级锁加锁：

​	线程在执行同步块之前，JVM会现在当前线程的栈帧中创建用于存储锁记录的空间，并将对象头中的MarkWord复制到锁记录中，官方称为Displaced Mark Word。然后线程尝试使用CAS将对象头中的MarkWord替换为指向锁记录的指针。如果成功，当前线程获得锁。如果失败，表示有其它线程竞争，当前线程便尝试使用自旋来获取锁。		

### 轻量级锁解锁：

​	轻量级解锁时，会使用原子的CAS操作将Displaced Mark Word 替换回对象头，如果成功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀为重量级锁。

图为两个线程同时争夺锁，导致锁膨胀。

![轻量级锁](https://user-gold-cdn.xitu.io/2019/10/8/16da8c153150b99e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 3、重量锁	

​	由于自旋会消耗CPU，为了避免无用的自旋，一旦锁升级为重量级锁，就不会再恢复到轻量级锁状态。当锁处于这个状态，其它线程试图获取锁，都会被阻塞住。

