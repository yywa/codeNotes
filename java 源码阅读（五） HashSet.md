# java 源码阅读（五）HashSet

HashSet是一个无序，允许空值，不允许重复值的集合类。非线程安全类。

## 继承/实现

![img](C:\Users\Admin\AppData\Local\Temp\企业微信截图_15697407933881.png)
## 构造函数

```java
private transient HashMap<E,Object> map;
private static final Object PRESENT = new Object();
```
### HashSet()

```java
public HashSet() {
    map = new HashMap<>();
}
```

可以看出，hashSet的底层实现是一个HashMap

### HashSet(int)

```java
public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}
```

### HashSet(int,float)

```java
public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}
```

### HashSet(int,float,boolean)

```java
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

专属于linkedHashSet的构造函数。

## 方法源码解读

### add(E)

```java
public boolean add(E e) {
  	//在map中添加了一个key为E，value为PRESENT的对象。
    return map.put(e, PRESENT)==null;
}
```

### remove(E)

```java
public boolean remove(Object o) {
  	//直接调用HashMap的remove方法
    return map.remove(o)==PRESENT;
}
```

### contains(Object)

```java
public boolean contains(Object o) {
  	//直接调用HashMap的containsKey()方法
    return map.containsKey(o);
}
```

源码中都是直接调用HashMap的方法，比较简单。