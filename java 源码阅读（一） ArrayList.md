# java 源码阅读（二） ArrayList

ArrayList是一种变长集合类，基于数组实现。ArrayList允许空值和重复元素。当往ArrayList中添加的元素数量超过底层数量时，会进行扩容。ArrayList实现了RandomAccess接口，所以可以保证在O（1）复杂度下完成随机查找操作。是一个非线程安全类，并发环境下，会出现错误。

## 实现/继承的类和接口

> extends AbstractList<E>  implements List<E>, RandomAccess, Cloneable, java.io.Serializable

## 构造函数

```java
transient Object[] elementData;
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
private static final Object[] EMPTY_ELEMENTDATA = {};
private static final int DEFAULT_CAPACITY = 10;
private int size;
protected transient int modCount = 0;
```

### 1.ArrayList()

```java
public ArrayList() {
  this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

初始化了一个空的数组对象。

### 2.ArrayList(int initialCapacity)

```java
  public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
```

如果参数值大于0，则初始化一个长度为**initialCapacity**的数组。否则初始化一个空数组。为负时抛出异常。

## add(E)

新建一个ArrayList对象，然后插入数据，进入到**add(E)**方法.

```java
public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
```

此时size为0，进入到**ensureCapacityInternal**()方法。

```java
private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }
```

**minCapacity =10**，进入到**ensureExplicitCapacity()**。

```java
private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```

**modCount = 1**，minCapacity - elementData.length > 0 成立，进入**grow()**方法

```java
 private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

newCapacity = 0，小于minCapacity，newCapacity = minCapacity，然后进入**Arrays.copyOf()**方法返回一个新的数组，这个数组长度为newCapacity  ，也就是10。

返回**add()**方法，elementData[size++] = e ，然后直接返回true。添加一个元素成功。

## add(int,E)

初始化一个长度为10的数组，然后向指定索引插入元素。

```java
public static void main(String[] args) {
   ArrayList arrayList = new ArrayList(10);
   arrayList.add(0);
   arrayList.add(10);
   arrayList.add(1, 2);
}
```
进入**add(int,E)**方法

```java
public void add(int index, E element) {
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    elementData[index] = element;
    size++;
}
```

进入**rangeCheckForAdd(index);**

```java
private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

size为2，验证通过。进入 **ensureCapacityInternal(size + 1);**

然后通过System.arraycopy(elementData, index, elementData, index + 1,size - index);方法，将后面的数据后移一位。

elementData[index] = element，将指定位置的数据变成要插入的数据。

 ![1569393154(1)](C:\Users\Admin\Desktop\1569393154(1).jpg)

## 扩容机制

```java
public class StringTest {
    public static void main(String[] args) {
        ArrayList arrayList = new ArrayList(10);
        for (int i = 0; i < 11; i++) {
            arrayList.add(10);
        }
        System.out.println(arrayList.size());
        System.out.println(getCapacity(arrayList));
    }

    public static Integer getCapacity(ArrayList list) {
        Integer length = 0;
        Class<? extends ArrayList> aClass = list.getClass();
        Field f;
        try {
            f = aClass.getDeclaredField("elementData");
            f.setAccessible(true);
            Object[] o = (Object[]) f.get(list);
            length = o.length;
        } catch (NoSuchFieldException | IllegalAccessException e) {
            e.printStackTrace();
        }
        return length;
    }
}

```

>11
>
>15

最后arrayList的长度为11，容量为15。

核心部分为：

```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

if (minCapacity - elementData.length > 0)   条件成立，此时就进入grow方法进行扩容。

```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

**int newCapacity = oldCapacity + (oldCapacity >> 1);**我们可以看出，新的数组大小是原来的1.5倍。

## remove(int)

```java
  public E remove(int index) {
    	//检查边界
        rangeCheck(index);
		
        modCount++;
    	//将原来位置的数据赋值。
        E oldValue = elementData(index);
		//计算需要移动的数据长度
        int numMoved = size - index - 1;
        if (numMoved > 0)
          	//进行数组的拷贝
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
    	//将最后一个索引位置上的数据置空。
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
```

## fastRemove(int)

快速删除，并没有进行边界检查等操作。在调用的方法内部会有校验。

```java
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

## remove(Object)

代码逻辑比较简单，此处不再赘述

```java
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
```

 ![企业微信截图_15693957167641](C:\Users\Admin\Desktop\企业微信截图_15693957167641.png)

  ## trimToSize()

arrayList删除大量元素后，会有一部分内存空间被空闲，我们可以调用该方法，将数组容量进行缩小，从而节省空间。

```java
public void trimToSize() {
    modCount++;
    if (size < elementData.length) {
        elementData = (size == 0)
          ? EMPTY_ELEMENTDATA
          : Arrays.copyOf(elementData, size);
    }
}
```
例子:

```java
public class StringTest {
    public static void main(String[] args) {
        ArrayList arrayList = new ArrayList(10);
        for (int i = 0; i < 200; i++) {
            arrayList.add(10);
        }
        System.out.println(getCapacity(arrayList));
        for (int i = 199; i > 100; i--) {
            arrayList.remove(i);
        }
        System.out.println(getCapacity(arrayList));
        arrayList.trimToSize();
        System.out.println(getCapacity(arrayList));
    }

    public static Integer getCapacity(ArrayList list) {
        Integer length = 0;
        Class<? extends ArrayList> aClass = list.getClass();
        Field f;
        try {
            f = aClass.getDeclaredField("elementData");
            f.setAccessible(true);
            Object[] o = (Object[]) f.get(list);
            length = o.length;
        } catch (NoSuchFieldException | IllegalAccessException e) {
            e.printStackTrace();
        }
        return length;
    }
}
```

>244
>244
>101

