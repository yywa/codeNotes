# java 源码阅读（三）LinkedList

LinkedList底层采用的是双向链表结构，支持空值和重复值。无法向ArrayList那样进行扩容，存储元素时，需要额外的空间存储前驱和后继的引用。LinkedList在链表头部和尾部的插入效率比较高，但在指定位置进行插入时，效率一般。操作复杂度为O(N)。LinkedList是非线程安全的集合类。

## 继承/实现



![img](C:\Users\Admin\AppData\Local\Temp\1569478560(1).jpg)

## 构造函数

```java
transient int size = 0;
transient Node<E> first;
transient Node<E> last;
```

```java
//内部类，双向链表
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

###LinkedList()

```java
public LinkedList() {
}
```
###LinkedList(Collection<? extends E>)
```java
public LinkedList(Collection<? extends E> c) {
     this();
     addAll(c);
}
```
## 方法源码解读

### get()

```java
public E get(int index) {
  	//用来验证当前index是否在size内
    checkElementIndex(index);
  	//返回结果
    return node(index).item;
}
```



```java
private void checkElementIndex(int index) {
    if (!isElementIndex(index))
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

```java
private boolean isElementIndex(int index) {
    return index >= 0 && index < size;
}
```

```java
Node<E> node(int index) {
    // assert isElementIndex(index);
	//这里是二分查找。
    if (index < (size >> 1)) {
      	//通过头结点，一直遍历后继结点到index位置。
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
      	//通过尾结点，一直遍历前驱结点到index位置。
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

### add()

```java
public boolean add(E e) {
    linkLast(e);
    return true;
}
```

```java
void linkLast(E e) {
  	//将最后一个结点赋给l
    final Node<E> l = last;
  	//新建一个Node，它的前驱结点是l
    final Node<E> newNode = new Node<>(l, e, null);
  	//将newNode的值赋给尾结点
    last = newNode;
    if (l == null)
      	//如果l为空，表示这是一个空的linkedList，将newNode赋给头结点
        first = newNode;
    else
      	//否则，将l的后继结点赋值为newNode
        l.next = newNode;
    size++;
    modCount++;
}
```

## add(int,E)

```java
public void add(int index, E element) {
  	//检验index是否在size内。
    checkPositionIndex(index);
		
    if (index == size)
      	//如果index==size，直接从尾部进行添加
        linkLast(element);
    else
      	//从index位置开始添加
        linkBefore(element, node(index));
}
```

```java
private void checkPositionIndex(int index) {
    if (!isPositionIndex(index))
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

```java
private boolean isPositionIndex(int index) {
    return index >= 0 && index <= size;
}
```

```java
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
  	//将succ的前驱结点赋值给pred
    final Node<E> pred = succ.prev;
  	//定义一个新的结点，它的前驱结点是pred，后继结点是succ
    final Node<E> newNode = new Node<>(pred, e, succ);
  	//将newNode赋值给succ的前驱结点
    succ.prev = newNode;
  	//验证当前链表是否为空
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

## remove()

```java
public E remove() {
    return removeFirst();
}
```

```java
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}
```

```java
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
```

代码比较简单，因此没有加注释。默认删除头结点，将第二个结点置为头结点。

## remove(int)

```java
public E remove(int index) {
  	//校验index
    checkElementIndex(index);
    return unlink(node(index));
}
```

```java
E unlink(Node<E> x) {
    // assert x != null;
  	//将删除的结点，后继结点，前驱结点取出，
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;
	//判断是否是头结点，修改后继结点
    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }
	//判断是否是尾结点，修改前驱结点
    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }
	
    x.item = null;
    size--;
    modCount++;
    return element;
}
```

## remove(Ojbect)

根据Object找到对应的index，然后进行删除。

```java
public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```