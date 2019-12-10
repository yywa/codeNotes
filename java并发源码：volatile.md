# java并发源码：volatile

## volatile的定义：

​	java编程语言运行线程访问共享变量，为了确保共享变量能被准确和一致地更新，线程应该确保通过排它锁单独获得这个变量。如果一个变量被声明成volatile，java线程内存模型确保所有线程看到这个变量的值是一致的。（可见性）

## volatile的作用：

### 1.保证变量在多线程中的可见性

​	为了提高处理速度，处理器不直接和内存进行通信，而是先将内存中的数据读到工作内存中后再进行操作，但操作完不知道何时会回写到内存中。

​	如果对有volatile声明的变量进行写操作，JVM会向处理器发送Lock前缀指令，将这个变量的值，回写到主内存中。

​	为了保证其它线程中的缓存一致，每个处理器经过嗅探在总线上传播的数据来检查自己缓存的值是否过期，如果发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行置为无效。当处理器再次进行修改操作时，会重新从主内存中读取数据。

##### 实现原则：

​	1.Lock前缀指令会引起处理器缓存回写到内存中。

​	2.一个处理器的缓存回写到内存中会导致其它处理器的缓存无效。

```java
public class VolatileTest {
    static volatile int x = 0;
    public static void increase() {
        x++;
    }

    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            for (int i = 0; i < 50000; i++) {
                increase();
            }
        }).start();
        new Thread(() -> {
            for (int i = 0; i < 50000; i++) {
                increase();
            }
        }).start();
        TimeUnit.SECONDS.sleep(5);
        System.out.println(x);
    }
}
```

输出`72867` 。

可以看出，volatile保证了在多线程中变量的可见性，但不保证原子性。

通过javap反编译出来的汇编。

> public class com.yyw.VolatileTest1 {
>   static volatile int x;
>
>   public com.yyw.VolatileTest1();
>     Code:
>        0: aload_0
>        1: invokespecial #1                  // Method java/lang/Object."<init>":()V
>        4: return
>
>   public static void increase();
>     Code:
>        0: getstatic     #2                  // Field x:I
>        3: iconst_1
>        4: iadd
>        5: putstatic     #2                  // Field x:I
>        8: return
>
>   public static void main(java.lang.String[]) throws java.lang.InterruptedException;
>     Code:
>        0: new           #3                  // class java/lang/Thread
>        3: dup
>        4: invokedynamic #4,  0              // InvokeDynamic #0:run:()Ljava/lang/Runnable;
>        9: invokespecial #5                  // Method java/lang/Thread."<init>":(Ljava/lang/Runnable;)V
>       12: invokevirtual #6                  // Method java/lang/Thread.start:()V
>       15: new           #3                  // class java/lang/Thread
>       18: dup
>       19: invokedynamic #7,  0              // InvokeDynamic #1:run:()Ljava/lang/Runnable;
>       24: invokespecial #5                  // Method java/lang/Thread."<init>":(Ljava/lang/Runnable;)V
>       27: invokevirtual #6                  // Method java/lang/Thread.start:()V
>       30: getstatic     #8                  // Field java/util/concurrent/TimeUnit.SECONDS:Ljava/util/concurrent/TimeUnit;
>       33: ldc2_w        #9                  // long 5l
>       36: invokevirtual #11                 // Method java/util/concurrent/TimeUnit.sleep:(J)V
>       39: getstatic     #12                 // Field java/lang/System.out:Ljava/io/PrintStream;
>       42: getstatic     #2                  // Field x:I
>       45: invokevirtual #13                 // Method java/io/PrintStream.println:(I)V
>       48: return
>
>   static {};
>     Code:
>        0: iconst_0
>        1: putstatic     #2                  // Field x:I
>        4: return
> }
>
> 

​	从编译出来的结果，我们可以看到，increase方法在Class文件中由四条字节码指令构成：

​	当getstatic指令把x的值取到操作栈顶时，volatile保证了x的值在此时是正确的，但是在执行iconst_1、i++指令时，其它线程可能已经把x的值加大了。此时x的值变成了过期的数据，putstatic指令执行后就可能把较小的值同步回主内存中。

### 2.禁止指令重排序

经典的DCL单例模式：为什么要检验两次INSTANCE==NULL？

```java
public class Singleton {
    private volatile static Singleton INSTANCE;

    public static Singleton getInstance() {
      	//1
        if (INSTANCE == null) {
            synchronized (Singleton.class) {
                if (INSTANCE == null) {
                  	//2
                    INSTANCE = new Singleton();
                }
            }
        }
        return INSTANCE;
    }
}
```

  new Singleton()，并不是一个原子操作，分为三步，

①.在内存中开辟一片内存区域

②.在这片内存区域中执行构造函数，实例化对象 

③.instance引用指向这片内存区域。

​	当A、B两个线程同时进入getInstance()方法时，A先执行，如果没有volatile修饰 ，上述操作可能被指令重排序为①->③->②，当线程执行到③，此时INSTANCE不再为null，若此时轮到B执行，B执行判断instance是否为NULL，直接返回INSTANCE，而此时INSTANCE指向的对象并没有完成初始化，会发生错误。

