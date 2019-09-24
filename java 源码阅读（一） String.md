# java 源码阅读（一） String

## String的属性：

char[] value; 

int hash;

## String的重要方法

### intern()

```
/**
* Returns a canonical representation for the string object.
* <p>
* A pool of strings, initially empty, is maintained privately by the
* class {@code String}.
* <p>
* When the intern method is invoked, if the pool already contains a
* string equal to this {@code String} object as determined by
* the {@link #equals(Object)} method, then the string from the pool is
* returned. Otherwise, this {@code String} object is added to the
* pool and a reference to this {@code String} object is returned.
* <p>
* It follows that for any two strings {@code s} and {@code t},
* {@code s.intern() == t.intern()} is {@code true}
* if and only if {@code s.equals(t)} is {@code true}.
* <p>
* All literal strings and string-valued constant expressions are
* interned. String literals are defined in section 3.10.5 of the
* <cite>The Java&trade; Language Specification</cite>.
*
* @return  a string that has the same contents as this string, but is
*          guaranteed to be from a pool of unique strings.
*/
public native String intern();
```

这是一个native方法，如果常量池中存在当前字符串，就会直接返回该字符串，如果常量池中没有此字符串，会将此字符串放入常量池后，再返回。

### hashcode()

```
/**
 * Returns a hash code for this string. The hash code for a
 * {@code String} object is computed as
 * <blockquote><pre>
 * s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
 * </pre></blockquote>
 * using {@code int} arithmetic, where {@code s[i]} is the
 * <i>i</i>th character of the string, {@code n} is the length of
 * the string, and {@code ^} indicates exponentiation.
 * (The hash value of the empty string is zero.)
 *
 * @return  a hash code value for this object.
 */
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```

这里他给了一个hashcode的计算公式：s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]。

这里有一个问题：**为什么hashcode乘子是31？**

31 可以被JVM优化，**31*i = (i << 5) -i**

**为什么不是2？**

因为2^5 = 31,hashcode的区间太小了，这样hashcode的冲突概率比较大。

## 例子一：

```
public static void main(String[] args) {
    String a = new String("ab");
    String b = "ab";
    String c = "a" + "b";
    System.out.println(a.equals(b));
    System.out.println(a.equals(c));
    System.out.println(b.equals(c));
    System.out.println(a == b);
    System.out.println(a == c);
    System.out.println(b == c);
}
```

```
true
true
true
false
false
true
```

**解释**

​	new String()无法在编译器确定。

​	String b = "ab" 可以在编译器确定，直接放入字符串常量池

​	String c = "a" + "b" 可以在编译器优化为 String c = "ab"

## 例子二：

```
public static void main(String[] args) {
    String a = new String("ab");
    String b = "ab";
    String c = "a" + new String("b");
    System.out.println(a.equals(b));
    System.out.println(a.equals(c));
    System.out.println(b.equals(c));
    System.out.println(a == b);
    System.out.println(a == c);
    System.out.println(b == c);
}
```

```
true
true
true
false
false
false
```

## 例子三：

```
public static void main(String[] args) {
    String a = "ab";
    String b = new String("ab");
    System.out.println(a == b);
    String c = b.intern();
    System.out.println(a == c);
}
```

```
false
true
```

