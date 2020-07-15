# JAVA 序列化

## 什么是序列化

>**序列化**:把对象转换为字节序列的过程称为对象的序列化。
>
>**反序列化**:把字节序列恢复为对象的过程称为对象的反序列化。

​		再来看看Java中的序列化。

## Serializable

```java
public interface Serializable {
}
```

​	Java中定义了一个空的接口。实现该空接口，即可实现序列化。

​	Java中有**ObjectOutputStream**和**ObjectInputStream**。

先定义一个Student类

```java
public class Student  {

    private String name;
    private int age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

然后写一个测试类

```java
public class Test {
    public static void main(String[] args) {
        Student student = new Student();
        student.setAge(1);
        student.setName("测试");
        //将对象写到文件中
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("test"))) {
            oos.writeObject(student);
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
        //从对象中读取文件
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("test"))) {
            Student o = (Student) ois.readObject();
            System.out.println(o.getAge());
            System.out.println(o.getName());
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

执行结果

>
>java.io.NotSerializableException: serial.Student
>	at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1184)
>	at java.io.ObjectOutputStream.writeObject(ObjectOutputStream.java:348)
>	at serial.Test.main(Test.java:17)
>java.io.WriteAbortedException: writing aborted; java.io.NotSerializableException: serial.Student
>	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1577)
>	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:431)
>	at serial.Test.main(Test.java:25)
>Caused by: java.io.NotSerializableException: serial.Student
>	at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1184)
>	at java.io.ObjectOutputStream.writeObject(ObjectOutputStream.java:348)
>	at serial.Test.main(Test.java:17)
>
>Process finished with exit code 0

可以看到，Student接口没有实现Serializable接口。修改Student类。

```java
public class Student implements Serializable {
	private static final long serialVersionUID = 7583950439381176616L;
    private String name;
    private int age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

再次执行Test类的Main方法

>1
>测试

可以看到，已经完成了正常的序列化和反序列化过程。

## 那么serialVersionUID有什么作用呢。

对Student类稍作修改

```java
private static final long serialVersionUID = 1L;
```

然后修改Test类

```java
public class Test {
    public static void main(String[] args) {
        //从对象中读取文件
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("test"))) {
            Student o = (Student) ois.readObject();
            System.out.println(o.getAge());
            System.out.println(o.getName());
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

运行后

> java.io.InvalidClassException: serial.Student; local class incompatible: stream classdesc serialVersionUID = 7583950439381176616, local class serialVersionUID = 1

可以看到，修改了serialVersionUID后，原来已经序列化后的内容，无法再反序列化。

## 注意事项：

static和transient修饰的字段是不可以被序列化的。

static不被序列化是因为他是属于类的属性，不是对象的属性，那么为什么transient修饰的也不可以被序列化呢？

在Student中添加b属性，并添加get/set方法。

```java
private transient String b;
```

对应的，也修改Test方法

```java
public static void main(String[] args) {
    Student student = new Student();
    student.setAge(1);
    student.setName("测试");
    student.setB("123");
    //将对象写到文件中
    try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("test"))) {
        oos.writeObject(student);
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
    //从对象中读取文件
    try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("test"))) {
        Student o = (Student) ois.readObject();
        System.out.println(o.getAge());
        System.out.println(o.getName());
        System.out.println(o.getB());
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
}
```

>1
>测试
>null

可以看到transient修饰的属性，拿到是空的。

深入源码ObjectStreamClass#getDefaultSerialFields

```java
/**
     * Returns array of ObjectStreamFields corresponding to all non-static
     * non-transient fields declared by given class.  Each ObjectStreamField
     * contains a Field object for the field it represents.  If no default
     * serializable fields exist, NO_FIELDS is returned.
     */
/**
	返回对应于给定类声明的所有非静态非瞬态字段的ObjectStreamFields数组。每个ObjectStreamField都包含一个代表该字段的Field对象。如果没有默认的可序列化字段存在，则返回NO_FIELDS。
*/
private static ObjectStreamField[] getDefaultSerialFields(Class<?> cl) {
    Field[] clFields = cl.getDeclaredFields();
    ArrayList<ObjectStreamField> list = new ArrayList<>();
    int mask = Modifier.STATIC | Modifier.TRANSIENT;

    for (int i = 0; i < clFields.length; i++) {
        if ((clFields[i].getModifiers() & mask) == 0) {
            list.add(new ObjectStreamField(clFields[i], false, true));
        }
    }
    int size = list.size();
    return (size == 0) ? NO_FIELDS :
        list.toArray(new ObjectStreamField[size]);
}
```

有没有办法可以实现transient 的反序列化呢？

我们对Student类再次改造

```java
public class Student implements Serializable {

    private static final long serialVersionUID = 7583950439381176616L;
    private String name;
    private int age;
    private transient String b;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getB() {
        return b;
    }

    public void setB(String b) {
        this.b = b;
    }

    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();
        out.writeObject(b);
    } 

    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject();
        b = (String) in.readObject();
    }
}
```

可以看到，新增了writeObject方法和readObject方法。在序列化和反序列化的时候，会调用writeObject和readObject方法。

执行Test，

>1
>测试
>123

可以看到，用transient修饰的变量，被反序列化出来。