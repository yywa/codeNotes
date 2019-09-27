# java 源码阅读（四）Vector

Vector是一种变长集合类，基于数组实现。ArrayList允许空值和重复元素。当往Vector中添加的元素数量超过底层数量时，会进行扩容。Vector实现了RandomAccess接口，所以可以保证在O（1）复杂度下完成随机查找操作。是一个线程安全类。

## 继承/实现

![img](C:\Users\Admin\AppData\Local\Temp\企业微信截图_15695655755377.png)
## 构造函数

```java
protected Object[] elementData;
protected int elementCount;
//扩容时增加的数组长度，默认为0
protected int capacityIncrement;
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

### Vector()

```java
public Vector() {
    this(10);
}
```

### Vector(int)

```java
public Vector(int initialCapacity) {
    this(initialCapacity, 0);
}
```

### Vector(int,int)

```java
public Vector(int initialCapacity, int capacityIncrement) {
    super();
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    this.elementData = new Object[initialCapacity];
    this.capacityIncrement = capacityIncrement;
}
```

构造函数比较简单，不再赘述。

## 方法源码解读

### add(E)

```java
public synchronized boolean add(E e) {
    modCount++;
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = e;
    return true;
}
```

```java
private void ensureCapacityHelper(int minCapacity) {
    // overflow-conscious code
  	//验证是否超出数组大小，然后进行扩容
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
  	//如果capacityIncrement没有主动赋值，默认为0.扩容为10。
    int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                     capacityIncrement : oldCapacity);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
  	//判断是否超出最大值
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

```java
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

### add(int,E)

```java
public void add(int index, E element) {
    insertElementAt(element, index);
}
```

```java
public synchronized void insertElementAt(E obj, int index) {
    modCount++;
    if (index > elementCount) {
        throw new ArrayIndexOutOfBoundsException(index
                                                 + " > " + elementCount);
    }
    ensureCapacityHelper(elementCount + 1);
    System.arraycopy(elementData, index, elementData, index + 1, elementCount - index);
    elementData[index] = obj;
    elementCount++;
}
```

基本流程与arrayList一致。

### get(int)

```java
public synchronized E get(int index) {
    if (index >= elementCount)
        throw new ArrayIndexOutOfBoundsException(index);

    return elementData(index);
}
```

```java
E elementData(int index) {
    return (E) elementData[index];
}
```

代码很简单。

### remove(int)

```java
public synchronized E remove(int index) {
    modCount++;
  	//验证index是否超出边界
    if (index >= elementCount)
        throw new ArrayIndexOutOfBoundsException(index);
    E oldValue = elementData(index);
	//取出index位置的数据，然后拷贝数组，将末尾数组元素置为空
    int numMoved = elementCount - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--elementCount] = null; // Let gc do its work

    return oldValue;
}
```

### remove(Object)

```java
public boolean remove(Object o) {
    return removeElement(o);
}
```

```java
public synchronized boolean removeElement(Object obj) {
    modCount++;
  	//确定obj的index
    int i = indexOf(obj);
    if (i >= 0) {
        removeElementAt(i);
        return true;
    }
    return false;
}
```
```java
 public int indexOf(Object o) {
        return indexOf(o, 0);
  }
```
```java
public synchronized int indexOf(Object o, int index) {
  	//对null进行了处理，然后循环遍历。直到匹配到o为止。
    if (o == null) {
        for (int i = index ; i < elementCount ; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = index ; i < elementCount ; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

```java
public synchronized void removeElementAt(int index) {
    modCount++;
  	//边界校验
    if (index >= elementCount) {
        throw new ArrayIndexOutOfBoundsException(index + " >= " +
                                                 elementCount);
    }
    else if (index < 0) {
        throw new ArrayIndexOutOfBoundsException(index);
    }
    int j = elementCount - index - 1;
    if (j > 0) {
        System.arraycopy(elementData, index + 1, elementData, index, j);
    }
    elementCount--;
    elementData[elementCount] = null; /* to let gc do its work */
}
```
流程基本与ArrayList一致。

看完源码，发现vector保证线程安全是靠着Synchronized关键字。下一篇，我们讲解Synchronized的源码