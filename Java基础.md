[TOC]



## Java基础

## 一、Integer

### 方法解析

Integer是int基本类型对应的包装类型，基本类型与其对应的包装类型之间的赋值使用自动装箱与拆箱完成。

```java
Integer x= 2; 		//装箱 调用了Integer.valueOf(2);
int y = x; 			//拆箱 调用了x.intValue();
```

##### valueOf()

实现简单，就是先判断值是否在IntegerCache中，存在直接返回缓冲池的内容。返回一个表示指定整型值的整数实例。如果不需要新的Integer实例，则通常应优先使用此方法，而不使用构造函数Integer(int)，因为通过缓存频繁请求的值，此方法可能会显著提高空间和时间性能。

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

##### equals(Object obj)

将此对象与指定的对象（obj）进行比较。当且仅当参数不为空且为包含与该对象相同整型值的整数对象时，结果为真。

```java
public boolean equals(Object obj) {
        if (obj instanceof Integer) {
            return value == ((Integer)obj).intValue();
        }
        return false;
    }
```

##### parseInt(String s, int radix)

根据参数radix（进制2~36之间）将String类型转换成int基本类型。

```java
		int result = 0;
        boolean negative = false;
        int i = 0, len = s.length();
        int limit = -Integer.MAX_VALUE;
        int multmin;
        int digit;

        if (len > 0) {
            char firstChar = s.charAt(0);
            //判断第一个字符是否合法；
            if (firstChar < '0') { // Possible leading "+" or "-"
                if (firstChar == '-') {
                    negative = true;
                    limit = Integer.MIN_VALUE;
                } else if (firstChar != '+')
                    throw NumberFormatException.forInputString(s);

                if (len == 1) // Cannot have lone "+" or "-"
                    throw NumberFormatException.forInputString(s);
                i++;
            }
            multmin = limit / radix;
            while (i < len) {
                // Accumulating negatively avoids surprises near MAX_VALUE
                digit = Character.digit(s.charAt(i++),radix);
                if (digit < 0) {
                    throw NumberFormatException.forInputString(s);
                }
                if (result < multmin) {
                    throw NumberFormatException.forInputString(s);
                }
                result *= radix;
                if (result < limit + digit) {
                    throw NumberFormatException.forInputString(s);
                }
                result -= digit;
            }
        } else {
            throw NumberFormatException.forInputString(s);
        }
        return negative ? result : -result;
```

##### getInteger(String nm)

确定具有指定名称的系统属性的整数值。第一个参数被视为系统属性的名称。可以通过java.lang.System.getProperty(java.lang.String)方法访问系统属性。然后，使用decode支持的语法将此属性的字符串值解释为一个整数值，并返回一个表示该值的整数对象。如果没有具有指定名称的属性，如果指定名称为空或null，或者如果属性没有正确的数字格式，则返回null。



##### highestOneBit(int i)

待

##### numberOfLeadingZeros(int i)

##### reverseBytes(int i)



### 内部类IntegerCache

doc：

Cache to support the object identity semantics of autoboxing for values  between -128 and 127 (inclusive) as required by JLS. The cache is initialized on  first usage. The size of the cache may be controlled by the  `-XX:AutoBoxCacheMax=<size>` option. During VM initialization,  java.lang.Integer.IntegerCache.high property may be set and saved in the private  system properties in the sun.misc.VM class.



new Integer(123)与Integer.valueOf(123)的区别在于：

●new Integer(123)每次都会创建一个对象

●Integer.valueOf(123)会查看缓冲池是否存在值为123的对象，存在则获取该对象引用，不存在则创建新的对象

```java
Integer x = new Integer(123);
Integer y = new Integer(123);
system.out.printf(x==y);		//false，两个对象不同
Integer z = Integer.valueOf(123);
Integer k = Integer.valueOf(123);
system.out.printf(z==k);		//true，获取到同一个对象
```

在Java8中，IntegerCache的默认大小为-128~127。

```java
private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1); //与下面创建数组长度有关
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;

            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);

            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }

        private IntegerCache() {}
    }
```

## 二、String

### 概述

String 被声明为 final，因此它不可被继承。(Integer 等包装类也不能被继承）

在 Java 8 中，String 内部使用 char 数组存储数据。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
}
```

在 Java 9 之后，String 类的实现改用 byte 数组存储字符串，同时使用 `coder` 来标识使用了哪种编码。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final byte[] value;

    /** The identifier of the encoding used to encode the bytes in {@code value}. */
    private final byte coder;
}
```

value 数组被声明为 final，这意味着 value 数组初始化之后就不能再引用其它数组。并且 String 内部没有改变 value 数组的方法，因此可以保证 String 不可变。

### 不可变的好处

**1.可以缓存hash值**

因为 String 的 hash 值经常被使用，例如 String 用做 HashMap 的 key。不可变的特性可以使得 hash 值也不可变，因此只需要进行一次计算。

**2.String Pool的需要**

如果一个 String 对象已经被创建过了，那么就会从 String Pool 中取得引用。只有 String 是不可变的，才可能使用 String Pool。

**3.安全性**

String 经常作为参数，String 不可变性可以保证参数不可变。例如在作为网络连接参数的情况下如果 String 是可变的，那么在网络连接过程中，String 被改变，改变 String 的那一方以为现在连接的是其它主机，而实际情况却不一定是。

**4.线程安全**

String 不可变性天生具备线程安全，可以在多个线程中安全地使用。

### 方法解析

##### equals(Object anObject)方法

将此字符串与指定的对象进行比较。当且仅当参数不为空且是表示与此对象相同字符序列的字符串对象时，结果为真。

```java
public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof String) {
            String anotherString = (String)anObject;
            int n = value.length;
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }
```

**栈区存引用和基本类型，不能存对象，而堆区存对象。==是比较地址，equals()比较对象内容。**

(1) **String str1 = "abcd"的实现过程**：首先栈区创建str引用，**然后在String池（独立于栈和堆而存在，存储不可变量）中寻找其指向的内容为"abcd"的对象，如果String池中没有，则创建一个，然后str指向String池中的对象，如果有，则直接将str1指向"abcd""；**如果后来又定义了字符串变量 str2 =  "abcd",则直接将str2引用指向String池中已经存在的“abcd”，不再重新创建对象；当str1进行了赋值（str1=“abc”），则str1将不再指向"abcd"，而是重新指String池中的"abc"，此时如果定义String str3 = "abc",进行str1 ==  str3操作，返回值为true，因为他们的值一样，地址一样，但是如果内容为"abc"的str1进行了字符串的+连接str1 =  str1+"d"；此时str1指向的是在堆中新建的内容为"abcd"的对象，即此时进行str1==str2，返回值false，因为地址不一样。

(2) **String str3 = new String("abcd")的实现过程：直接在堆中创建对象**。如果后来又有String str4 = new  String("abcd")，str4不会指向之前的对象，而是重新创建一个对象并指向它，所以如果此时进行str3==str4返回值是false，因为两个对象的地址不一样，如果是str3.equals(str4)，返回true,因为内容相同。

##### index(int ch, int fromIndex)

返回指定字符（ch）第一次出现的字符串中的索引，在指定的索引处（fromIndex）开始搜索。返回指定字符第一次出现的字符串中的索引，在指定的索引处开始搜索。

```java
public int indexOf(int ch, int fromIndex) {
        final int max = value.length;
        if (fromIndex < 0) {
            fromIndex = 0;
        } else if (fromIndex >= max) {
            // Note: fromIndex might be near -1>>>1.
            return -1;
        }

        if (ch < Character.MIN_SUPPLEMENTARY_CODE_POINT) {
            // handle most cases here (ch is a BMP code point or a
            // negative value (invalid code point))
            final char[] value = this.value;
            for (int i = fromIndex; i < max; i++) {
                if (value[i] == ch) {
                    return i;
                }
            }
            return -1;
        } else {
            return indexOfSupplementary(ch, fromIndex);	//处理带有附加字符的indexOf()
        }
    }
```

##### lastIndexOf(int ch, int fromIndex)

返回指定字符最后一次出现的字符串中的索引，从指定的索引开始向后搜索。

```java
public int lastIndexOf(int ch, int fromIndex) {
        if (ch < Character.MIN_SUPPLEMENTARY_CODE_POINT) {
            // handle most cases here (ch is a BMP code point or a
            // negative value (invalid code point))
            final char[] value = this.value;
            int i = Math.min(fromIndex, value.length - 1);
            for (; i >= 0; i--) {
                if (value[i] == ch) {
                    return i;
                }
            }
            return -1;
        } else {
            return lastIndexOfSupplementary(ch, fromIndex);
        }
    }
```

##### startsWith(String prefix, int toffset)

测试从指定索引处（toffset）开始的此字符串的子字符串是否以指定的前缀开始。

```java
public boolean startsWith(String prefix, int toffset) {
        char ta[] = value;
        int to = toffset;
        char pa[] = prefix.value;
        int po = 0;
        int pc = prefix.value.length;
        // Note: toffset might be near -1>>>1.
        if ((toffset < 0) || (toffset > value.length - pc)) {
            return false;
        }
        while (--pc >= 0) {
            if (ta[to++] != pa[po++]) {
                return false;
            }
        }
        return true;
    }
```

##### trim()

返回一个字符串，该字符串的值为该字符串，删除了任何头部和尾部空格。如果这个字符串对象表示一个空字符序列，或者这个字符串对象表示的字符序列的第一个和最后一个字符的代码都大于'\u005Cu0020'(空格字符)，那么返回对这个字符串对象的引用。否则，如果字符串中没有代码大于'\u005Cu0020'的字符，则返回一个表示空字符串的字符串对象。此方法可用于从字符串的开始和结束部分修剪空白(如上定义)。

```java
public String trim() {
        int len = value.length;
        int st = 0;
        char[] val = value;    /* avoid getfield opcode */

        while ((st < len) && (val[st] <= ' ')) {
            st++;
        }
        while ((st < len) && (val[len - 1] <= ' ')) {
            len--;
        }
        return ((st > 0) || (len < value.length)) ? substring(st, len) : this;
    }
```

##### join(CharSequence delimiter,Iterable<? extends CharSequence> elements)

返回一个由CharSequence元素的副本和指定分隔符的副本组成的新字符串。

```java
public static String join(CharSequence delimiter,
            Iterable<? extends CharSequence> elements) {
        Objects.requireNonNull(delimiter);
        Objects.requireNonNull(elements);
        StringJoiner joiner = new StringJoiner(delimiter);
        for (CharSequence cs: elements) {
            joiner.add(cs);
        }
        return joiner.toString();
    }
	 //example:
	 //List<String> strings = new LinkedList<>();
     //strings.add("Java");strings.add("is");
     //strings.add("cool");
     //String message = String.join("-", strings);
     //message returned is: "Java-is-cool"

```

##### 重写join(CharSequence delimiter,Iterable<? extends CharSequence> elements)

同上方法，只是在字符串的头部和尾部添加了新的固定字符组成新字符串。

```java
public static String join(CharSequence delimiter,Iterable<? extends CharSequence> elements) {
		Objects.requireNonNull(delimiter);
		Objects.requireNonNull(elements);
		StringJoiner joiner = new StringJoiner(delimiter, "{", "}");//可以替换前缀和后缀
		for (CharSequence chs : elements) {
			joiner.add(chs);
		}
		return joiner.toString();
	}
```

##### contentEquals(CharSequence cs)

判断该字符串与指定的字符串（实现了CharSequence接口的类）是否相等。如果cs是String，那该方法等同于equals(cs)。注意：指定字符串是StringBuffer则同步该方法。

```java
 public boolean contentEquals(CharSequence cs) {
        // Argument is a StringBuffer, StringBuilder
        if (cs instanceof AbstractStringBuilder) {
            if (cs instanceof StringBuffer) {
                synchronized(cs) {
                   return nonSyncContentEquals((AbstractStringBuilder)cs);
                }
            } else {
                return nonSyncContentEquals((AbstractStringBuilder)cs);
            }
        }
        // Argument is a String
        if (cs instanceof String) {
            return equals(cs);
        }
        // Argument is a generic CharSequence
        char v1[] = value;
        int n = v1.length;
        if (n != cs.length()) {
            return false;
        }
        for (int i = 0; i < n; i++) {
            if (v1[i] != cs.charAt(i)) {
                return false;
            }
        }
        return true;
    }
```

##### equalsIgnoreCase(String anotherString)

将此字符串与另一个字符串进行比较，忽略大小写注意事项。如果两个字符串长度相同，并且两个字符串中的对应字符都是相等的忽略大小写，则认为它们是相等的忽略大小写。

```java
 public boolean equalsIgnoreCase(String anotherString) {
        return (this == anotherString) ? true
                : (anotherString != null)
                && (anotherString.value.length == value.length)
                && regionMatches(true, 0, anotherString, 0, value.length);
    }
```

##### regionMatches(boolean ignoreCase, int toffset,String other, int ooffset, int len)

测试两个字符串区域是否相等。

```java
 public boolean regionMatches(boolean ignoreCase, int toffset,
            String other, int ooffset, int len) {
        char ta[] = value;
        int to = toffset;
        char pa[] = other.value;
        int po = ooffset;
        // Note: toffset, ooffset, or len might be near -1>>>1.
        if ((ooffset < 0) || (toffset < 0)
                || (toffset > (long)value.length - len)
                || (ooffset > (long)other.value.length - len)) {
            return false;
        }
        while (len-- > 0) {
            char c1 = ta[to++];
            char c2 = pa[po++];
            if (c1 == c2) {
                continue;
            }
            if (ignoreCase) {
                // If characters don't match but case may be ignored,
                // try converting both characters to uppercase.
                // If the results match, then the comparison scan should
                // continue.
                char u1 = Character.toUpperCase(c1);
                char u2 = Character.toUpperCase(c2);
                if (u1 == u2) {
                    continue;
                }
                // Unfortunately, conversion to uppercase does not work properly
                // for the Georgian alphabet, which has strange rules about case
                // conversion.  So we need to make one last check before
                // exiting.
                if (Character.toLowerCase(u1) == Character.toLowerCase(u2)) {
                    continue;
                }
            }
            return false;
        }
        return true;
    }
```

##### split(String regex, int limit)

将这个字符串拆分为与给定正则表达式匹配的字符串。此方法返回的数组包含此字符串的每个子字符串，该子字符串由与给定表达式匹配的另一个子字符串终止，或由字符串的末尾终止。数组中的子字符串按照它们在该字符串中出现的顺序排列。如果表达式不匹配输入的任何部分，那么得到的数组只有一个元素，即这个字符串。

```java
public String[] split(String regex, int limit) {
        /* fastpath if the regex is a
         (1)one-char String and this character is not one of the
            RegEx's meta characters ".$|()[{^?*+\\", or
         (2)two-char String and the first char is the backslash and
            the second is not the ascii digit or ascii letter.
         */
        char ch = 0;
        if (((regex.value.length == 1 &&
             ".$|()[{^?*+\\".indexOf(ch = regex.charAt(0)) == -1) ||
             (regex.length() == 2 &&
              regex.charAt(0) == '\\' &&
              (((ch = regex.charAt(1))-'0')|('9'-ch)) < 0 &&
              ((ch-'a')|('z'-ch)) < 0 &&
              ((ch-'A')|('Z'-ch)) < 0)) &&
            (ch < Character.MIN_HIGH_SURROGATE ||
             ch > Character.MAX_LOW_SURROGATE))
        {
            int off = 0;
            int next = 0;
            boolean limited = limit > 0;
            ArrayList<String> list = new ArrayList<>();
            while ((next = indexOf(ch, off)) != -1) {
                if (!limited || list.size() < limit - 1) {
                    list.add(substring(off, next));
                    off = next + 1;
                } else {    // last one
                    //assert (list.size() == limit - 1);
                    list.add(substring(off, value.length));
                    off = value.length;
                    break;
                }
            }
            // If no match was found, return this
            if (off == 0)
                return new String[]{this};

            // Add remaining segment
            if (!limited || list.size() < limit)
                list.add(substring(off, value.length));

            // Construct result
            int resultSize = list.size();
            if (limit == 0) {
                while (resultSize > 0 && list.get(resultSize - 1).length() == 0) {
                    resultSize--;
                }
            }
            String[] result = new String[resultSize];
            return list.subList(0, resultSize).toArray(result);
        }
        return Pattern.compile(regex).split(this, limit);
    }
```



### String, StringBuffer and StringBuilder

**1. 可变性**

- String 不可变
- StringBuffer 和 StringBuilder 可变

**2. 线程安全**

- String 不可变，因此是线程安全的
- StringBuilder 不是线程安全的
- StringBuffer 是线程安全的，内部使用 synchronized 进行同步

### new String("abc")

使用这种方式一共会创建两个字符串对象（前提是 String Pool 中还没有 "abc" 字符串对象）。

- "abc" 属于字符串字面量，因此编译时期会在 String Pool 中创建一个字符串对象，指向这个 "abc" 字符串字面量；
- 而使用 new 的方式会在堆中创建一个字符串对象。

创建一个测试类，其 main 方法中使用这种方式来创建字符串对象。

```
public class NewStringTest {
    public static void main(String[] args) {
        String s = new String("abc");
    }
}
```

使用 javap -verbose 进行反编译，得到以下内容：

```java
// ...
Constant pool:
// ...
   #2 = Class              #18            // java/lang/String
   #3 = String             #19            // abc
// ...
  #18 = Utf8               java/lang/String
  #19 = Utf8               abc
// ...

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=2, args_size=1
         0: new           #2                  // class java/lang/String
         3: dup
         4: ldc           #3                  // String abc
         6: invokespecial #4                  // Method java/lang/String."<init>":(Ljava/lang/String;)V
         9: astore_1
// ...
```

在 Constant Pool 中，#19 存储这字符串字面量 "abc"，#3 是 String Pool 的字符串对象，它指向 #19 这个字符串字面量。在 main 方法中，0: 行使用 new #2 在堆中创建一个字符串对象，并且使用 ldc #3 将 String  Pool 中的字符串对象作为 String 构造函数的参数。

以下是 String 构造函数的源码，可以看到，在将一个字符串对象作为另一个字符串对象的构造函数参数时，并不会完全复制 value 数组内容，而是都会指向同一个 value 数组。

```java
public String(String original) {
    this.value = original.value;
    this.hash = original.hash;
}
```

## 三、运算

### 参数传递

Java的参数是按值传递的形式传入方法中，不是按引用传递。



以下代码中 Dog dog 的 dog 是一个指针，存储的是对象的地址。在将一个参数传入一个方法时，本质上是将对象的地址以值的方式传递到形参中。

```java
public class Dog {

    String name;

    Dog(String name) {
        this.name = name;
    }

    String getName() {
        return this.name;
    }

    void setName(String name) {
        this.name = name;
    }

    String getObjectAddress() {
        return super.toString();
    }
}
```

在方法中改变对象的字段值会改变原对象该字段值，因为引用的是同一个对象。

```java
class PassByValueExample {
    public static void main(String[] args) {
        Dog dog = new Dog("A");
        func(dog);
        System.out.println(dog.getName());          // B
    }

    private static void func(Dog dog) {
        dog.setName("B");
    }
}
```

但是在方法中将指针引用了其它对象，那么此时方法里和方法外的两个指针指向了不同的对象，在一个指针改变其所指向对象的内容对另一个指针所指向的对象没有影响。

```java
public class PassByValueExample {
    public static void main(String[] args) {
        Dog dog = new Dog("A");
        System.out.println(dog.getObjectAddress()); // Dog@4554617c
        func(dog);
        System.out.println(dog.getObjectAddress()); // Dog@4554617c
        System.out.println(dog.getName());          // A
    }

    private static void func(Dog dog) {
        System.out.println(dog.getObjectAddress()); // Dog@4554617c
        dog = new Dog("B");
        System.out.println(dog.getObjectAddress()); // Dog@74a14482
        System.out.println(dog.getName());          // B
    }
}
```

### float 与 double

Java 不能隐式执行向下转型，因为这会使得精度降低。

1.1 字面量属于 double 类型，不能直接将 1.1 直接赋值给 float 变量，因为这是向下转型。

```java
// float f = 1.1;
```

1.1f 字面量才是 float 类型。

```java
float f = 1.1f;
```

### 隐式类型转换

因为字面量 1 是 int 类型，它比 short 类型精度要高，因此不能隐式地将 int 类型向下转型为 short 类型。

```java
short s1 = 1;
// s1 = s1 + 1;
```

但是使用 += 或者 ++ 运算符会执行隐式类型转换。

```java
s1 += 1;
s1++;
```

上面的语句相当于将 s1 + 1 的计算结果进行了向下转型：

```java
s1 = (short) (s1 + 1);
```

### switch

从 Java 7 开始，可以在 switch 条件判断语句中使用 String 对象。

```java
String s = "a";
switch (s) {
    case "a":
        System.out.println("aaa");
        break;
    case "b":
        System.out.println("bbb");
        break;
}
```

switch 不支持 long，是因为 switch 的设计初衷是对那些只有少数几个值的类型进行等值判断，如果值过于复杂，那么还是用 if 比较合适。

```java
// long x = 111;
// switch (x) { // Incompatible types. Found: 'long', required: 'char, byte, short, int, Character, Byte, Short, Integer, String, or an enum'
//     case 111:
//         System.out.println(111);
//         break;
//     case 222:
//         System.out.println(222);
//         break;
// }
```

## 四、关键字

### final

#### **1. 数据**

声明数据为常量，可以是编译时常量，也可以是在运行时被初始化后不能被改变的常量。

- 对于基本类型，final 使数值不变；
- 对于引用类型，final 使引用不变，也就不能引用其它对象，但是被引用的对象本身是可以修改的。

```java
final int x = 1;
// x = 2;  // cannot assign value to final variable 'x'
final A y = new A();
y.a = 1;
```

#### **2. 方法**

声明方法不能被子类重写。

private 方法隐式地被指定为 final，如果在子类中定义的方法和基类中的一个 private 方法签名相同，此时子类的方法不是重写基类方法，而是在子类中定义了一个新的方法。

#### **3. 类**

声明类不允许被继承。

## 

### static

#### **1. 静态变量**

- 静态变量：又称为类变量，也就是说这个变量属于类的，类所有的实例都共享静态变量，可以直接通过类名来访问它。静态变量在内存中只存在一份。
- 实例变量：每创建一个实例就会产生一个实例变量，它与该实例同生共死。

```java
public class A {

    private int x;         // 实例变量
    private static int y;  // 静态变量

    public static void main(String[] args) {
        // int x = A.x;  // Non-static field 'x' cannot be referenced from a static context
        A a = new A();
        int x = a.x;
        int y = A.y;
    }
}
```

#### **2. 静态方法**

静态方法在类加载的时候就存在了，它不依赖于任何实例。所以静态方法必须有实现，也就是说它不能是抽象方法。

```java
public abstract class A {
    public static void func1(){
    }
    // public abstract static void func2();  // Illegal combination of modifiers: 'abstract' and 'static'
}
```

只能访问所属类的静态字段和静态方法，方法中不能有 this 和 super 关键字，因此这两个关键字与具体对象关联。

```java
public class A {

    private static int x;
    private int y;

    public static void func1(){
        int a = x;
        // int b = y;  // Non-static field 'y' cannot be referenced from a static context
        // int b = this.y;     // 'A.this' cannot be referenced from a static context
    }
}
```

#### **3. 静态语句块**

静态语句块在类初始化时运行一次。

```java
public class A {
    static {
        System.out.println("123");
    }

    public static void main(String[] args) {
        A a1 = new A();
        A a2 = new A();
    }
}
123
```

#### **4. 静态内部类**

非静态内部类依赖于外部类的实例，也就是说需要先创建外部类实例，才能用这个实例去创建非静态内部类。而静态内部类不需要。

```java
public class OuterClass {

    class InnerClass {
    }

    static class StaticInnerClass {
    }

    public static void main(String[] args) {
        // InnerClass innerClass = new InnerClass(); // 'OuterClass.this' cannot be referenced from a static context
        OuterClass outerClass = new OuterClass();
        InnerClass innerClass = outerClass.new InnerClass();
        StaticInnerClass staticInnerClass = new StaticInnerClass();
    }
}
```

静态内部类不能访问外部类的非静态的变量和方法。

#### **5. 静态导包**

在使用静态变量和方法时不用再指明 ClassName，从而简化代码，但可读性大大降低。

```java
import static com.xxx.ClassName.*
```

#### **6. 初始化顺序**

静态变量和静态语句块优先于实例变量和普通语句块，静态变量和静态语句块的初始化顺序取决于它们在代码中的顺序。

```java
public static String staticField = "静态变量";
static {
    System.out.println("静态语句块");
}
public String field = "实例变量";
{
    System.out.println("普通语句块");
}
```

最后才是构造函数的初始化。

```java
public InitialOrderTest() {
    System.out.println("构造函数");
}
```

存在继承的情况下，初始化顺序为：

- 父类（静态变量、静态语句块）
- 子类（静态变量、静态语句块）
- 父类（实例变量、普通语句块）
- 父类（构造函数）
- 子类（实例变量、普通语句块）
- 子类（构造函数）

## 五、Object 通用方法

### 概览

```java
public native int hashCode()

public boolean equals(Object obj)

protected native Object clone() throws CloneNotSupportedException

public String toString()

public final native Class<?> getClass()

protected void finalize() throws Throwable {}

public final native void notify()

public final native void notifyAll()

public final native void wait(long timeout) throws InterruptedException

public final void wait(long timeout, int nanos) throws InterruptedException

public final void wait() throws InterruptedException
```

### equals()

```java
public boolean equals(Object obj) {
        return (this == obj);
}
```



#### 1.等价关系

两个对象具有等价关系，需要满足以下五个条件：

Ⅰ 自反性

```java
x.equals(x); // true
```

Ⅱ 对称性

```java
x.equals(y) == y.equals(x); // true
```

Ⅲ 传递性

```java
if (x.equals(y) && y.equals(z))
    x.equals(z); // true;
```

Ⅳ 一致性

多次调用 equals() 方法结果不变

```java
x.equals(y) == x.equals(y); // true
```

Ⅴ 与 null 的比较

对任何不是 null 的对象 x 调用 x.equals(null) 结果都为 false

```java
x.equals(null); // false;
```

#### 2. 等价与相等

- 对于基本类型，== 判断两个值是否相等，基本类型没有 equals() 方法。
- 对于引用类型，== 判断两个变量是否引用同一个对象，而 equals() 判断引用的对象是否等价。

```java
Integer x = new Integer(1);
Integer y = new Integer(1);
System.out.println(x.equals(y)); // true
System.out.println(x == y);      // false
```

#### 3. 实现

- 检查是否为同一个对象的引用，如果是直接返回 true；
- 检查是否是同一个类型，如果不是，直接返回 false；
- 将 Object 对象进行转型；
- 判断每个关键域是否相等。

```java
public class EqualExample {

    private int x;
    private int y;
    private int z;

    public EqualExample(int x, int y, int z) {
        this.x = x;
        this.y = y;
        this.z = z;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        EqualExample that = (EqualExample) o;

        if (x != that.x) return false;
        if (y != that.y) return false;
        return z == that.z;
    }
}
```

#### 遇到的问题

在类Integer中重写的equals（）方法是判断相同类型对象且值相等时就返回true，而在类Object中的equals（）方法是判断是否是同一个对象。

```java
public static void main(String[] args) {
		Object x = new Integer(1);
		Object y = new Integer(1);
		System.out.println(x == y);
		System.out.println(x.equals(y)); // y为2时false;但是y改为1则变成true，是还要比较值是否相等吗？还是说该equals调用的是Integer的：答 调用Integer的

    	//A是B的父类：向上转型
		A a = new B();
		a.print();

		Object m = new Object();
		Object n = new Object();
		System.out.println(m == n);
		System.out.println(m.equals(n));
	}
```

### hashCode()

#### 概述

hashCode() 返回哈希值，而 equals() 是用来判断两个对象是否等价。等价的两个对象散列值一定相同，但是散列值相同的两个对象不一定等价，这是因为计算哈希值具有随机性，两个值不同的对象可能计算出相同的哈希值。

在覆盖 equals() 方法时应当总是覆盖 hashCode() 方法，保证等价的两个对象哈希值也相等。

HashSet  和 HashMap 等集合类使用了 hashCode()  方法来计算对象应该存储的位置，因此要将对象添加到这些集合类中，需要让对应的类实现 hashCode()  方法。

下面的代码中，新建了两个等价的对象，并将它们添加到 HashSet 中。我们希望将这两个对象当成一样的，只在集合中添加一个对象。但是  EqualExample 没有实现 hashCode() 方法，因此这两个对象的哈希值是不同的，最终导致集合添加了两个等价的对象。

```java
EqualExample e1 = new EqualExample(1, 1, 1);
EqualExample e2 = new EqualExample(1, 1, 1);
System.out.println(e1.equals(e2)); // true
HashSet<EqualExample> set = new HashSet<>();
set.add(e1);
set.add(e2);
System.out.println(set.size());   // 2
```

理想的哈希函数应当具有均匀性，即不相等的对象应当均匀分布到所有可能的哈希值上。这就要求了哈希函数要把所有域的值都考虑进来。可以将每个域都当成 R 进制的某一位，然后组成一个 R 进制的整数。

R 一般取 31，因为它是一个奇素数，如果是偶数的话，当出现乘法溢出，信息就会丢失，因为与 2 相乘相当于向左移一位，最左边的位丢失。并且一个数与 31 相乘可以转换成移位和减法：`31*x == (x<<5)-x`，编译器会自动进行这个优化。	

```java
@Override
public int hashCode() {
    int result = 17;
    result = 31 * result + x;
    result = 31 * result + y;
    result = 31 * result + z;
    return result;
}
```

#### 遇到的问题

重写了hashCode（）方法后，必须重写equals（）方法，因为在HashMap的put方法内需要用到equals（）方法判断对象是否已存在，如果没有重写则会调用Object的equals（）方法。

```
public static void main(String[] args) {
		Example e1 = new Example(1, 1, 1);
		Example e2 = new Example(1, 1, 1);


		System.out.println(e1.hashCode());
		System.out.println(e2.hashCode());


		HashSet<Example> set = new HashSet<ObjectHashCode.Example>();

		set.add(e1);
		set.add(e2);

		System.out.println(set.size()); // hash值一样为啥是2？

	}

```

### toString()

默认返回 ToStringExample@4554617c 这种形式，其中 @ 后面的数值为散列码的无符号十六进制表示。

```java
public class ToStringExample {

    private int number;

    public ToStringExample(int number) {
	        this.number = number;
	}
}
```

```java
ToStringExample example = new ToStringExample(123);
System.out.println(example.toString());
```

结果：ToStringExample@4554617c

#### 源码分析

在Object类中的toString（）方法将调用类Integer的 ---------（）方法，将对象的hash值作参传入。

### clone()

####  cloneable

clone() 是 Object 的 protected 方法，它不是 public，一个类不显式去重写 clone()，其它类就不能直接去调用该类实例的 clone() 方法。

```java
public class CloneExample {
    private int a;
    private int b;
}
CloneExample e1 = new CloneExample();
// CloneExample e2 = e1.clone(); // 'clone()' has protected access in 'java.lang.Object'
```

重写 clone() 得到以下实现：

```java
public class CloneExample {
    private int a;
    private int b;

    @Override
    public CloneExample clone() throws CloneNotSupportedException {
        return (CloneExample)super.clone();
    }
}
CloneExample e1 = new CloneExample();
try {
    CloneExample e2 = e1.clone();
} catch (CloneNotSupportedException e) {
    e.printStackTrace();
}
java.lang.CloneNotSupportedException: CloneExample
```

以上抛出了 CloneNotSupportedException，这是因为 CloneExample 没有实现 Cloneable 接口。

应该注意的是，clone() 方法并不是 Cloneable 接口的方法，而是 Object 的一个 protected  方法。Cloneable 接口只是规定，如果一个类没有实现 Cloneable 接口又调用了 clone() 方法，就会抛出  CloneNotSupportedException。

```java
public class CloneExample implements Cloneable {
    private int a;
    private int b;

    @Override
    public Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
```

#### 浅拷贝

拷贝对象和原始对象的引用类型引用同一个对象。

```java
public class ShallowCloneExample implements Cloneable {

    private int[] arr;

    public ShallowCloneExample() {
        arr = new int[10];
        for (int i = 0; i < arr.length; i++) {
            arr[i] = i;
        }
    }

    public void set(int index, int value) {
        arr[index] = value;
    }

    public int get(int index) {
        return arr[index];
    }

    @Override
    protected ShallowCloneExample clone() throws CloneNotSupportedException {
        return (ShallowCloneExample) super.clone();
    }
}
public void main(String[] args){
    ShallowCloneExample e1 = new ShallowCloneExample();
	ShallowCloneExample e2 = null;
	try {
    		e2 = e1.clone();
	} catch (CloneNotSupportedException e) {
    	e.printStackTrace();
	}
	e1.set(2, 222);
	System.out.println(e2.get(2)); // 222
}
```

#### 深拷贝

拷贝对象和原始对象的引用类型引用不同对象。

```java
public class DeepCloneExample implements Cloneable {

    private int[] arr;

    public DeepCloneExample() {
        arr = new int[10];
        for (int i = 0; i < arr.length; i++) {
            arr[i] = i;
        }
    }

    public void set(int index, int value) {
        arr[index] = value;
    }

    public int get(int index) {
        return arr[index];
    }

    @Override
    protected DeepCloneExample clone() throws CloneNotSupportedException {
        DeepCloneExample result = (DeepCloneExample) super.clone();
        result.arr = new int[arr.length];
        for (int i = 0; i < arr.length; i++) {
            result.arr[i] = arr[i];
        }
        return result;
    }
}
public void main(String[] args){
	DeepCloneExample e1 = new DeepCloneExample();
	DeepCloneExample e2 = null;
	try {
  	  e2 = e1.clone();
	} catch (CloneNotSupportedException e) {
	    e.printStackTrace();
	}
	e1.set(2, 222);
	System.out.println(e2.get(2)); // 2
}
```

#### 深拷贝与浅拷贝的区别

```java
public class CloneTest {

	public static void main(String[] args) {
		ShallowCloneExample se1 = new ShallowCloneExample();
		ShallowCloneExample se2 = null;
		try {
			se2 = se1.clone();
		} catch (CloneNotSupportedException e) {
			e.printStackTrace();
		}
		// 克隆后使用equals比较两个对象（重写后的equals）
		System.out.println("shallow clone equals() :" + se1.equals(se2)); // true
		// 因浅拷贝指向同一个地址，修改值后会影响另外的对象
		se1.set(2, 222);
		System.out.println("shallow clone :" + se2.get(2)); // 222
		// e1和e2是两个不同对象（引用类型），值为相同的指向数据的地址
		System.out.println("shallow clone object :" + (se1 == se2)); // false
		// 克隆后的class类型相同
		System.out.println("shallow clone class :" + (se1.getClass() == se2.getClass())); 		  // true

		DeepCloneExample de1 = new DeepCloneExample();
		DeepCloneExample de2 = null;
		try {
			de2 = de1.clone();
		} catch (CloneNotSupportedException e) {
			e.printStackTrace();
		}
		// 克隆后使用equals比较两个对象（重写后的equals）
		System.out.println("deep clone equals() :" + de1.equals(de2)); // true
		// 因深拷贝是完全拷贝（全新的对象），修改值后不影响其他对象
		de1.set(2, 222);
		System.out.println("deep clone :" + de2.get(2)); // 2
		// de1和de2是两个不同对象（引用类型），值为两个不同的指向数据的地址
		System.out.println("deep clone object :" + (de1 == de2)); // false
		// 克隆后的class类型相同
		System.out.println("deep clone class :" + (de1.getClass() == de2.getClass())); 			// true

	}

	static class ShallowCloneExample implements Cloneable {

		private int[] arr;

		public ShallowCloneExample() {
			arr = new int[10];
			for (int i = 0; i < arr.length; i++) {
				arr[i] = i;
			}
		}

		public void set(int index, int value) {
			arr[index] = value;
		}

		public int get(int index) {
			return arr[index];
		}

		@Override
		protected ShallowCloneExample clone() throws CloneNotSupportedException {
			return (ShallowCloneExample) super.clone();
		}

		@Override
		public boolean equals(Object obj) {
			if (obj instanceof ShallowCloneExample) {
				ShallowCloneExample another = (ShallowCloneExample) obj;
				if (arr.length != another.arr.length) {
					return false;
				} else {
					for (int i = 0; i < arr.length; i++) {
						if (arr[i] != another.arr[i]) {
							return false;
						}
					}
					return true;
				}
			}
			return false;
		}
	}

	static class DeepCloneExample implements Cloneable {

		private int[] arr;

		public DeepCloneExample() {
			arr = new int[10];
			for (int i = 0; i < arr.length; i++) {
				arr[i] = i;
			}
		}

		public void set(int index, int value) {
			arr[index] = value;
		}

		public int get(int index) {
			return arr[index];
		}

		@Override
		protected DeepCloneExample clone() throws CloneNotSupportedException {
			DeepCloneExample result = (DeepCloneExample) super.clone();
			result.arr = new int[arr.length];
			for (int i = 0; i < arr.length; i++) {
				result.arr[i] = arr[i];
			}
			return result;
		}

		@Override
		public boolean equals(Object obj) {
			if (obj instanceof DeepCloneExample) {
				DeepCloneExample another = (DeepCloneExample) obj;
				if (arr.length != another.arr.length) {
					return false;
				} else {
					for (int i = 0; i < arr.length; i++) {
						if (arr[i] != another.arr[i]) {
							return false;
						}
					}
					return true;
				}
			}
			return false;
		}
	}
}
```



#### clone() 的替代方案

使用 clone() 方法来拷贝一个对象即复杂又有风险，它会抛出异常，并且还需要类型转换。Effective Java 书上讲到，最好不要去使用 clone()，可以使用拷贝构造函数或者拷贝工厂来拷贝一个对象。

```java
public class CloneConstructorExample {

    private int[] arr;

    public CloneConstructorExample() {
        arr = new int[10];
        for (int i = 0; i < arr.length; i++) {
            arr[i] = i;
        }
    }

    public CloneConstructorExample(CloneConstructorExample original) {
        arr = new int[original.arr.length];
        for (int i = 0; i < original.arr.length; i++) {
            arr[i] = original.arr[i];
        }
    }

    public void set(int index, int value) {
        arr[index] = value;
    }

    public int get(int index) {
        return arr[index];
    }
}
```

```java
CloneConstructorExample e1 = new CloneConstructorExample();
CloneConstructorExample e2 = new CloneConstructorExample(e1);
e1.set(2, 222);
System.out.println(e2.get(2)); // 2
```

## 六、继承

#### 访问权限

Java 中有三个访问权限修饰符：private、protected 以及 public，如果不加访问修饰符，表示包级可见。

可以对类或类中的成员（字段和方法）加上访问修饰符。

- 类可见表示其它类可以用这个类创建实例对象。
- 成员可见表示其它类可以用这个类的实例对象访问到该成员；

**protected 用于修饰成员，表示在继承体系中成员对于子类可见，但是这个访问修饰符对于类没有意义。外包可以通过继承来访问protected修饰的成员字段**

设计良好的模块会隐藏所有的实现细节，把它的 API 与它的实现清晰地隔离开来。模块之间只通过它们的 API 进行通信，一个模块不需要知道其他模块的内部工作情况，这个概念被称为信息隐藏或封装。因此访问权限应当尽可能地使每个类或者成员不被外界访问。

**如果子类的方法重写了父类的方法，那么子类中该方法的访问级别不允许低于父类的访问级别**。这是为了确保可以使用父类实例的地方都可以使用子类实例去代替，也就是确保满足里氏替换原则。

字段决不能是公有的，因为这么做的话就失去了对这个字段修改行为的控制，客户端可以对其随意修改。例如下面的例子中，AccessExample 拥有 id 公有字段，如果在某个时刻，我们想要使用 int 存储 id 字段，那么就需要修改所有的客户端代码。

```java
public class AccessExample {
    public String id;
}
```

可以使用公有的 getter 和 setter 方法来替换公有字段，这样的话就可以控制对字段的修改行为。

```java
public class AccessExample {

    private int id;

    public String getId() {
        return id + "";
    }

    public void setId(String id) {
        this.id = Integer.valueOf(id);
    }
}
```

但是也有例外，如果是包级私有的类或者私有的嵌套类，那么直接暴露成员不会有特别大的影响。

```java
public class AccessWithInnerClassExample {

    private class InnerClass {
        int x;
    }

    private InnerClass innerClass;

    public AccessWithInnerClassExample() {
        innerClass = new InnerClass();
    }

    public int getValue() {
        return innerClass.x;  // 直接访问
    }
}
```

##### 关于访问权限的小知识

在Method类中，访问权限用整型表示：

public=1    protected=4    default=0    private=2

#### 抽象类与接口

##### 1. 抽象类

抽象类和抽象方法都使用 abstract 关键字进行声明。如果一个类中包含抽象方法，那么这个类必须声明为抽象类。

抽象类和普通类最大的区别是，抽象类不能被实例化，只能被继承。

```java
public abstract class AbstractClassExample {

    protected int x;
    private int y;

    public abstract void func1();

    public void func2() {
        System.out.println("func2");
    }
}
```

```java
public class AbstractExtendClassExample extends AbstractClassExample {
    @Override
    public void func1() {
        System.out.println("func1");
    }
}
```



```java
// AbstractClassExample ac1 = new AbstractClassExample(); 
// 'AbstractClassExample' is abstract; cannot be instantiated
AbstractClassExample ac2 = new AbstractExtendClassExample();
ac2.func1();
```



##### 2. 接口

接口是抽象类的延伸，在 Java 8 之前，它可以看成是一个完全抽象的类，也就是说它不能有任何的方法实现。

从 Java 8 开始，接口也可以拥有默认的方法实现，这是因为不支持默认方法的接口的维护成本太高了。在 Java 8 之前，如果一个接口想要添加新的方法，那么要修改所有实现了该接口的类，让它们都实现新增的方法。

接口的成员（字段 + 方法）默认都是 public 的，并且不允许定义为 private 或者 protected。

接口的字段默认都是 static 和 final 的。

```java
public interface InterfaceExample {

    void func1();

    default void func2(){
        System.out.println("func2");
    }

    int x = 123;
    // int y;               // Variable 'y' might not have been initialized
    public int z = 0;       // Modifier 'public' is redundant for interface fields
    // private int k = 0;   // Modifier 'private' not allowed here
    // protected int l = 0; // Modifier 'protected' not allowed here
    // private void fun3(); // Modifier 'private' not allowed here
}
```

```java
public class InterfaceImplementExample implements InterfaceExample {
    @Override
    public void func1() {
        System.out.println("func1");
    }
}
```

```java
// InterfaceExample ie1 = new InterfaceExample(); 
// 'InterfaceExample' is abstract; cannot be instantiated
InterfaceExample ie2 = new InterfaceImplementExample();
ie2.func1();
System.out.println(InterfaceExample.x);
```



##### 3. 比较

- 从设计层面上看，抽象类提供了一种 IS-A 关系，需要满足里式替换原则，即子类对象必须能够替换掉所有父类对象。而接口更像是一种 LIKE-A 关系，它只是提供一种方法实现契约，并不要求接口和实现接口的类具有 IS-A 关系。
- 从使用上来看，一个类可以实现多个接口，但是不能继承多个抽象类。
- 接口的字段只能是 static 和 final 类型的，而抽象类的字段没有这种限制。
- 接口的成员只能是 public 的，而抽象类的成员可以有多种访问权限。

4. ##### 使用选择

使用接口：

- 需要让不相关的类都实现一个方法，例如不相关的类都可以实现 Comparable 接口中的 compareTo() 方法；
- 需要使用多重继承。

使用抽象类：

- 需要在几个相关的类中共享代码。
- 需要能控制继承来的成员的访问权限，而不是都为 public。
- 需要继承非静态和非常量字段。

在很多情况下，接口优先于抽象类。因为接口没有抽象类严格的类层次结构要求，可以灵活地为一个类添加行为。并且从 Java 8 开始，接口也可以有默认的方法实现，使得修改接口的成本也变的很低。

- [Abstract Methods and Classes](https://docs.oracle.com/javase/tutorial/java/IandI/abstract.html)
- [深入理解 abstract class 和 interface](https://www.ibm.com/developerworks/cn/java/l-javainterface-abstract/)
- [When to Use Abstract Class and Interface](https://dzone.com/articles/when-to-use-abstract-class-and-intreface)



#### super

- 访问父类的构造函数：可以使用 super()  函数访问父类的构造函数，从而委托父类完成一些初始化的工作。应该注意到，子类一定会调用父类的构造函数来完成初始化工作，一般是调用父类的默认构造函数，如果子类需要调用父类其它构造函数，那么就可以使用 super() 函数。
- 访问父类的成员：如果子类重写了父类的某个方法，可以通过使用 super 关键字来引用父类的方法实现。

```java
public class SuperExample {

    protected int x;
    protected int y;

    public SuperExample(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public void func() {
        System.out.println("SuperExample.func()");
    }
}
public class SuperExtendExample extends SuperExample {

    private int z;

    public SuperExtendExample(int x, int y, int z) {
        super(x, y);
        this.z = z;
    }

    @Override
    public void func() {
        super.func();
        System.out.println("SuperExtendExample.func()");
    }
}
```

```java
SuperExample e = new SuperExtendExample(1, 2, 3);
e.func();
SuperExample.func()
SuperExtendExample.func()
```

[Using the Keyword super](https://docs.oracle.com/javase/tutorial/java/IandI/super.html)

#### 重写与重载

##### 1. 重写（Override）

存在于继承体系中，指子类实现了一个与父类在方法声明上完全相同的一个方法。

为了满足里式替换原则，重写有以下三个限制：

- **子类方法的访问权限必须大于等于父类方法**；
- **子类方法的返回类型必须是父类方法返回类型或为其子类型。**
- **子类方法抛出的异常类型必须是父类抛出异常类型或为其子类型。**

使用 @Override 注解，可以让编译器帮忙检查是否满足上面的三个限制条件。

下面的示例中，SubClass 为 SuperClass 的子类，SubClass 重写了 SuperClass 的 func() 方法。其中：

- 子类方法访问权限为 public，大于父类的 protected。
- 子类的返回类型为 ArrayList，是父类返回类型 List 的子类。
- 子类抛出的异常类型为 Exception，是父类抛出异常 Throwable 的子类。
- 子类重写方法使用 @Override 注解，从而让编译器自动检查是否满足限制条件。

```java
class SuperClass {
    protected List<Integer> func() throws Throwable {
        return new ArrayList<>();
    }
}

class SubClass extends SuperClass {
    @Override
    public ArrayList<Integer> func() throws Exception {
        return new ArrayList<>();
    }
}
```

**在调用一个方法时，先从本类中查找看是否有对应的方法，如果没有再到父类中查看，看是否从父类继承来。否则就要对参数进行转型，转成父类之后看是否有对应的方法。总的来说，方法调用的优先级为：**

- **this.func(this)**
- **super.func(this)**
- **this.func(super)**
- **super.func(super)**

```java
/*
    A
    |
    B
    |
    C
    |
    D
 */


class A {

    public void show(A obj) {
        System.out.println("A.show(A)");
    }

    public void show(C obj) {
        System.out.println("A.show(C)");
    }
}

class B extends A {

    @Override
    public void show(A obj) {
        System.out.println("B.show(A)");
    }
}

class C extends B {
}

class D extends C {
}
public static void main(String[] args) {

    A a = new A();
    B b = new B();
    C c = new C();
    D d = new D();

    // 在 A 中存在 show(A obj)，直接调用
    a.show(a); // A.show(A)
    // 在 A 中不存在 show(B obj)，将 B 转型成其父类 A
    a.show(b); // A.show(A)
    // 在 B 中存在从 A 继承来的 show(C obj)，直接调用
    b.show(c); // A.show(C)
    // 在 B 中不存在 show(D obj)，但是存在从 A 继承来的 show(C obj)，将 D 转型成其父类 C
    b.show(d); // A.show(C)

    // 引用的还是 B 对象，所以 ba 和 b 的调用结果一样
    A ba = new B();
    ba.show(c); // A.show(C)
    ba.show(d); // A.show(C)
}
```

##### 2. 重载（Overload）

存在于同一个类中，指一个方法与已经存在的方法名称上相同，但是参数类型、个数、顺序至少有一个不同。

应该注意的是，返回值不同，其它都相同不算是重载。

##### 遇到的问题

子类能够重写父类的静态方法吗？（与jvm有关？类存放的内存区域与静态方法的内存区域不同？）

答：不能。静态方法属于类，是不能被重写，故而也不能实现多态。

[为什么不能重写静态方法？](https://blog.csdn.net/dawn_after_dark/article/details/74357049?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase)



## 七、反射

每个类都有一个   **Class**   对象，包含了与类有关的信息。当编译一个新类时，会产生一个同名的 .class 文件，该文件内容保存着 Class 对象。

类加载相当于 Class 对象的加载，类在第一次使用时才动态加载到 JVM 中。也可以使用 `Class.forName("com.mysql.jdbc.Driver")` 这种方式来控制类的加载，该方法会返回一个 Class 对象。

反射可以提供运行时的类信息，并且这个类可以在运行时才加载进来，甚至在编译时期该类的 .class 不存在也可以加载进来。

Class 和 java.lang.reflect 一起对反射提供了支持，java.lang.reflect 类库主要包含了以下三个类：

- **Field**  ：可以使用 get() 和 set() 方法读取和修改 Field 对象关联的字段；
- **Method**  ：可以使用 invoke() 方法调用与 Method 对象关联的方法；
- **Constructor**  ：可以用 Constructor 的 newInstance() 创建新的对象。

### 反射的优点

- **可扩展性**：应用程序可以利用全限定名创建可扩展对象的实例，来使用来自外部的用户自定义类。
- **类浏览器和可视化开发环境**：一个类浏览器需要可以枚举类的成员。可视化开发环境（如 IDE）可以从利用反射中可用的类型信息中受益，以帮助程序员编写正确的代码。
- **调试器和测试工具**： 调试器需要能够检查一个类里的私有成员。测试工具可以利用反射来自动地调用类里定义的可被发现的 API 定义，以确保一组测试中有较高的代码覆盖率。

### 反射的缺点

尽管反射非常强大，但也不能滥用。如果一个功能可以不用反射完成，那么最好就不用。在我们使用反射技术时，下面几条内容应该牢记于心。

- **性能开销**：反射涉及了动态类型的解析，所以 JVM 无法对这些代码进行优化。因此，反射操作的效率要比那些非反射操作低得多。我们应该避免在经常被执行的代码或对性能要求很高的程序中使用反射。
- **安全限制**：使用反射技术要求程序必须在一个没有安全限制的环境中运行。如果一个程序必须在有安全限制的环境中运行，如 Applet，那么这就是个问题了。
- **内部暴露**：由于反射允许代码执行一些在正常情况下不被允许的操作（比如访问私有的属性和方法），所以使用反射可能会导致意料之外的副作用，这可能导致代码功能失调并破坏可移植性。反射代码破坏了抽象性，因此当平台发生改变的时候，代码的行为就有可能也随着变化。

- [Trail: The Reflection API](https://docs.oracle.com/javase/tutorial/reflect/index.html)
- [深入解析 Java 反射（1）- 基础](http://www.sczyh30.com/posts/Java/java-reflection-1/)

### 深入解析

Java 反射主要提供以下功能：

- 在运行时判断任意一个对象所属的类；
- 在运行时构造任意一个类的对象；
- 在运行时判断任意一个类所具有的成员变量和方法（通过反射甚至可以调用private方法）；
- 在运行时调用任意一个对象的方法

**重点是运行时而不是编译时**

#### 反射的基本运用

获取class对象的方法有三种：

1. 使用Class类的forName静态方法：

   ```java
   public static Class<?> forName(String className)
    
   //比如在 JDBC 开发中常用此方法加载数据库驱动:
   java
   Class.forName(driver);
   ```

2. 直接获取某一个对象的class，比如：

   ```java
   Class<?> klass = int.class;
   Class<?> classInt = Integer.TYPE;
   ```

3. 调用某个对象的getClass（）方法，比如：

   ```java
   StringBuilder str = new StringBuilder("123");
   Class<?> klass = str.getClass();
   ```

#### 利用反射创建数组

数组在Java里是比较特殊的一种类型，它可以赋值给一个Object Reference。下面我们看一看利用反射创建数组的例子：

```java
public static void testArray() throws ClassNotFoundException {
        Class<?> cls = Class.forName("java.lang.String");
        Object array = Array.newInstance(cls,25);
        //往数组里添加内容
        Array.set(array,0,"hello");
        Array.set(array,1,"Java");
        Array.set(array,2,"fuck");
        Array.set(array,3,"Scala");
        Array.set(array,4,"Clojure");
        //获取某一项的内容
        System.out.println(Array.get(array,3));
    }
```

其中的Array类为java.lang.reflect.Array类。我们通过Array.newInstance()创建数组对象，它的原型是:

```java
public static Object newInstance(Class<?> componentType, int length)
        throws NegativeArraySizeException {
        return newArray(componentType, length);
    }
```

而 `newArray` 方法是一个 native 方法，它在 HotSpot JVM 里的具体实现我们后边再研究，这里先把源码贴出来：

```java
private static native Object newArray(Class<?> componentType, int length)
        throws NegativeArraySizeException;
```

源码目录：`openjdk\hotspot\src\share\vm\runtime\reflection.cpp`

```c++
arrayOop Reflection::reflect_new_array(oop element_mirror, jint length, TRAPS) {
  if (element_mirror == NULL) {
    THROW_0(vmSymbols::java_lang_NullPointerException());
  }
  if (length < 0) {
    THROW_0(vmSymbols::java_lang_NegativeArraySizeException());
  }
  if (java_lang_Class::is_primitive(element_mirror)) {
    Klass* tak = basic_type_mirror_to_arrayklass(element_mirror, CHECK_NULL);
    return TypeArrayKlass::cast(tak)->allocate(length, THREAD);
  } else {
    Klass* k = java_lang_Class::as_Klass(element_mirror);
    if (k->oop_is_array() && ArrayKlass::cast(k)->dimension() >= MAX_DIM) {
      THROW_0(vmSymbols::java_lang_IllegalArgumentException());
    }
    return oopFactory::new_objArray(k, length, THREAD);
  }
}
```

另外，Array 类的 `set` 和 `get` 方法都为 native 方法，在 HotSpot JVM 里分别对应 `Reflection::array_set` 和 `Reflection::array_get` 方法，这里就不详细解析了。

#### 解析invoke方法

invoke的实现：

```java
@CallerSensitive
public Object invoke(Object obj, Object... args)
    throws IllegalAccessException, IllegalArgumentException,
       InvocationTargetException
{
    if (!override) {
        if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
            Class<?> caller = Reflection.getCallerClass();
            checkAccess(caller, clazz, obj, modifiers);
        }
    }
    MethodAccessor ma = methodAccessor;             // read volatile
    if (ma == null) {
        ma = acquireMethodAccessor();
    }
    return ma.invoke(obj, args);
}
```

根据invoke方法的实现，可以将其分为以下几步：

##### 1.权限检查

在invoke方法中首先判断AccessibleObject的override属性的值。AccessibleObject类是Field、Method和Constructor对象的基类。**它提供了将反射的对象标记为在使用时取消默认 Java 语言访问控制检查的能力**（猜测：反射能够使用私有权限的关键）。当Fields、Methods或Constructors用于设置或获取字段、调用方法或创建和初始化类的新实例时，将分别执行访问检查——针对public、default(包)访问、protected和private。

override的值默认是false,表示需要权限调用规则，调用方法时需要检查权限;我们也可以用setAccessible方法设置为true,若override的值为true，表示忽略权限规则，调用方法时无需检查权限（也就是说可以调用任意的private方法，违反了封装）。

如果override属性为默认值false，则进行进一步的权限检查：

（1）首先用Reflection.quickCheckMemberAccess(clazz,  modifiers)方法检查方法是否为public，如果是的话跳出本步；如果不是public方法，那么用Reflection.getCallerClass()方法获取调用这个方法的Class对象，这是一个native方法:

```java
@CallerSensitive
    public static native Class<?> getCallerClass();
```

在OpenJDK的源码中找到此方法的JNI入口(Reflection.c):

```java
JNIEXPORT jclass JNICALL Java_sun_reflect_Reflection_getCallerClass__
(JNIEnv *env, jclass unused)
{
    return JVM_GetCallerClass(env, JVM_CALLER_DEPTH);
}
```

其中JVM_GetCallerClass的源码如下，有兴趣的可以研究一下(位于jvm.cpp):

```c++
JVM_ENTRY(jclass, JVM_GetCallerClass(JNIEnv* env, int depth))
  JVMWrapper("JVM_GetCallerClass");

  // Pre-JDK 8 and early builds of JDK 8 don't have a CallerSensitive annotation; or
  // sun.reflect.Reflection.getCallerClass with a depth parameter is provided
  // temporarily for existing code to use until a replacement API is defined.
  if (SystemDictionary::reflect_CallerSensitive_klass() == NULL || depth != JVM_CALLER_DEPTH) {
    Klass* k = thread->security_get_caller_class(depth);
    return (k == NULL) ? NULL : (jclass) JNIHandles::make_local(env, k->java_mirror());
  }

  // Getting the class of the caller frame.
  //
  // The call stack at this point looks something like this:
  //
  // [0] [ @CallerSensitive public sun.reflect.Reflection.getCallerClass ]
  // [1] [ @CallerSensitive API.method                                   ]
  // [.] [ (skipped intermediate frames)                                 ]
  // [n] [ caller                                                        ]
  vframeStream vfst(thread);
  // Cf. LibraryCallKit::inline_native_Reflection_getCallerClass
  for (int n = 0; !vfst.at_end(); vfst.security_next(), n++) {
    Method* m = vfst.method();
    assert(m != NULL, "sanity");
    switch (n) {
    case 0:
      // This must only be called from Reflection.getCallerClass
      if (m->intrinsic_id() != vmIntrinsics::_getCallerClass) {
        THROW_MSG_NULL(vmSymbols::java_lang_InternalError(), "JVM_GetCallerClass must only be called from Reflection.getCallerClass");
      }
      // fall-through
    case 1:
      // Frame 0 and 1 must be caller sensitive.
      if (!m->caller_sensitive()) {
        THROW_MSG_NULL(vmSymbols::java_lang_InternalError(), err_msg("CallerSensitive annotation expected at frame %d", n));
      }
      break;
    default:
      if (!m->is_ignored_by_security_stack_walk()) {
        // We have reached the desired frame; return the holder class.
        return (jclass) JNIHandles::make_local(env, m->method_holder()->java_mirror());
      }
      break;
    }
  }
  return NULL;
JVM_END
```

获取了这个Class对象caller后用checkAccess方法做一次快速的权限校验，其实现为:

（疑问：caller是什么？猜测：是调用该方法的类，因为要判断该类与被调用方法的类之间的关系，即访问控制）

```java
volatile Object securityCheckCache;

    void checkAccess(Class<?> caller, Class<?> clazz, Object obj, int modifiers)
        throws IllegalAccessException
    {
        if (caller == clazz) {  // 快速校验
            return;             // 权限通过校验
        }
        Object cache = securityCheckCache;  // read volatile
        Class<?> targetClass = clazz;
        if (obj != null
            && Modifier.isProtected(modifiers)
            && ((targetClass = obj.getClass()) != clazz)) {
            // Must match a 2-list of { caller, targetClass }.
            if (cache instanceof Class[]) {
                Class<?>[] cache2 = (Class<?>[]) cache;
                if (cache2[1] == targetClass &&
                    cache2[0] == caller) {
                    return;     // ACCESS IS OK
                }
                // (Test cache[1] first since range check for [1]
                // subsumes range check for [0].)
            }
        } else if (cache == caller) {
            // Non-protected case (or obj.class == this.clazz).
            return;             // ACCESS IS OK
        }

        // If no return, fall through to the slow path.
        slowCheckMemberAccess(caller, clazz, obj, modifiers, targetClass);
    }
```



首先先执行一次快速校验，一旦调用方法的Class正确则权限检查通过。
若未通过，则创建一个缓存，中间再进行一堆检查（比如检验是否为protected属性）。
如果上面的所有权限检查都未通过，那么将执行更详细的检查，其实现为：

```java
// Keep all this slow stuff out of line:
void slowCheckMemberAccess(Class<?> caller, Class<?> clazz, Object obj, int modifiers,
                           Class<?> targetClass)
    throws IllegalAccessException
{
    Reflection.ensureMemberAccess(caller, clazz, obj, modifiers);

    // Success: Update the cache.
    Object cache = ((targetClass == clazz)
                    ? caller
                    : new Class<?>[] { caller, targetClass });

    // Note:  The two cache elements are not volatile,
    // but they are effectively final.  The Java memory model
    // guarantees that the initializing stores for the cache
    // elements will occur before the volatile write.
    securityCheckCache = cache;         // write volatile
}
```



大体意思就是，用Reflection.ensureMemberAccess方法继续检查权限，若检查通过就更新缓存，这样下一次同一个类调用同一个方法时就不用执行权限检查了，这是一种简单的缓存机制。由于JMM的happens-before规则能够保证缓存初始化能够在写缓存之前发生，因此两个cache不需要声明为volatile。
到这里，前期的权限检查工作就结束了。如果没有通过检查则会抛出异常，如果通过了检查则会到下一步。

##### 2.调用MethodAccessor的invoke方法

Method.invoke()实际上并不是自己实现的反射调用逻辑，而是委托给sun.reflect.MethodAccessor来处理。
首先要了解Method对象的基本构成，每个Java方法有且只有一个Method对象作为root，它相当于根对象，对用户不可见。当我们创建Method对象时，我们代码中获得的Method对象都相当于它的副本（或引用）。root对象持有一个MethodAccessor对象，所以所有获取到的Method对象都共享这一个MethodAccessor对象，因此必须保证它在内存中的可见性。root对象其声明及注释为：

```java
private volatile MethodAccessor methodAccessor;
// For sharing of MethodAccessors. This branching structure is
// currently only two levels deep (i.e., one root Method and
// potentially many Method objects pointing to it.)
//
// If this branching structure would ever contain cycles, deadlocks can
// occur in annotation code.
private Method  root;
```

那么MethodAccessor到底是个啥玩意呢？

```java
/** This interface provides the declaration for
    java.lang.reflect.Method.invoke(). Each Method object is
    configured with a (possibly dynamically-generated) class which
    implements this interface.
*/
	public interface MethodAccessor {
    /** Matches specification in {@link java.lang.reflect.Method} */
    public Object invoke(Object obj, Object[] args)
        throws IllegalArgumentException, InvocationTargetException;
}
```

可以看到MethodAccessor是一个接口，定义了invoke方法。分析其Usage可得它的具体实现类有:

- sun.reflect.DelegatingMethodAccessorImpl
- sun.reflect.MethodAccessorImpl
- sun.reflect.NativeMethodAccessorImpl

第一次调用一个Java方法对应的Method对象的invoke()方法之前，实现调用逻辑的MethodAccessor对象还没有创建；等第一次调用时才新创建MethodAccessor并更新给root，然后调用MethodAccessor.invoke()完成反射调用：

```java
// NOTE that there is no synchronization used here. It is correct
// (though not efficient) to generate more than one MethodAccessor
// for a given Method. However, avoiding synchronization will
// probably make the implementation more scalable.
private MethodAccessor acquireMethodAccessor() {
    // First check to see if one has been created yet, and take it
    // if so
    MethodAccessor tmp = null;
    if (root != null) tmp = root.getMethodAccessor();
    if (tmp != null) {
        methodAccessor = tmp;
    } else {
        // Otherwise fabricate one and propagate it up to the root
        tmp = reflectionFactory.newMethodAccessor(this);
        setMethodAccessor(tmp);
    }

    return tmp;
}
```

可以看到methodAccessor实例由reflectionFactory对象操控生成，它在AccessibleObject下的声明如下:

```java
// Reflection factory used by subclasses for creating field,
// method, and constructor accessors. Note that this is called
// very early in the bootstrapping process.
static final ReflectionFactory reflectionFactory =
    AccessController.doPrivileged(
        new sun.reflect.ReflectionFactory.GetReflectionFactoryAction());
```

再研究一下sun.reflect.ReflectionFactory类的源码：

```java
public class ReflectionFactory {

    private static boolean initted = false;
    private static Permission reflectionFactoryAccessPerm
        = new RuntimePermission("reflectionFactoryAccess");
    private static ReflectionFactory soleInstance = new ReflectionFactory();
    // Provides access to package-private mechanisms in java.lang.reflect
    private static volatile LangReflectAccess langReflectAccess;

    // 这里设计得非常巧妙
    // "Inflation" mechanism. Loading bytecodes to implement
    // Method.invoke() and Constructor.newInstance() currently costs
    // 3-4x more than an invocation via native code for the first
    // invocation (though subsequent invocations have been benchmarked
    // to be over 20x faster). Unfortunately this cost increases
    // startup time for certain applications that use reflection
    // intensively (but only once per class) to bootstrap themselves.
    // To avoid this penalty we reuse the existing JVM entry points
    // for the first few invocations of Methods and Constructors and
    // then switch to the bytecode-based implementations.
    //
    // Package-private to be accessible to NativeMethodAccessorImpl
    // and NativeConstructorAccessorImpl
    private static boolean noInflation        = false;
    private static int     inflationThreshold = 15;// 调用次数阈值，超过此值生成一个专用的MethodAccessor实现类，以后调用该java方法的反射调用就会使用java版本。from jdk1.4

    //......

	//这是生成MethodAccessor的方法
    public MethodAccessor newMethodAccessor(Method method) {
        checkInitted();

        if (noInflation && !ReflectUtil.isVMAnonymousClass(method.getDeclaringClass())) {
            return new MethodAccessorGenerator().
                generateMethod(method.getDeclaringClass(),
                               method.getName(),
                               method.getParameterTypes(),
                               method.getReturnType(),
                               method.getExceptionTypes(),
                               method.getModifiers());
        } else {
            NativeMethodAccessorImpl acc =
                new NativeMethodAccessorImpl(method);
            DelegatingMethodAccessorImpl res =
                new DelegatingMethodAccessorImpl(acc);
            acc.setParent(res);
            return res;
        }
    }

    //......

    /** We have to defer full initialization of this class until after
    the static initializer is run since java.lang.reflect.Method's
    static initializer (more properly, that for
    java.lang.reflect.AccessibleObject) causes this class's to be
    run, before the system properties are set up. */
    private static void checkInitted() {
        if (initted) return;
        AccessController.doPrivileged(
            new PrivilegedAction<Void>() {
                public Void run() {
                    // Tests to ensure the system properties table is fully
                    // initialized. This is needed because reflection code is
                    // called very early in the initialization process (before
                    // command-line arguments have been parsed and therefore
                    // these user-settable properties installed.) We assume that
                    // if System.out is non-null then the System class has been
                    // fully initialized and that the bulk of the startup code
                    // has been run.

                    if (System.out == null) {
                        // java.lang.System not yet fully initialized
                        return null;
                    }

                    String val = System.getProperty("sun.reflect.noInflation");
                    if (val != null && val.equals("true")) {
                        noInflation = true;
                    }

                    val = System.getProperty("sun.reflect.inflationThreshold");
                    if (val != null) {
                        try {
                            inflationThreshold = Integer.parseInt(val);
                        } catch (NumberFormatException e) {
                            throw new RuntimeException("Unable to parse property sun.reflect.inflationThreshold", e);
                        }
                    }

                    initted = true;
                    return null;
                }
            });
    }
}
```

观察前面的声明部分的注释，我们可以发现一些有趣的东西。就像注释里说的，实际的MethodAccessor实现有两个版本，一个是Java版本，一个是native版本，两者各有特点。初次启动时Method.invoke()和Constructor.newInstance()方法采用native方法要比Java方法快3-4倍，而启动后native方法又要消耗额外的性能而慢于Java方法。也就是说，Java实现的版本在初始化时需要较多时间，但长久来说性能较好；native版本正好相反，启动时相对较快，但运行时间长了之后速度就比不过Java版了。这是HotSpot的优化方式带来的性能特性，同时也是许多虚拟机的共同点：跨越native边界会对优化有阻碍作用，它就像个黑箱一样让虚拟机难以分析也将其内联，于是运行时间长了之后反而是托管版本的代码更快些。

为了尽可能地减少性能损耗，HotSpot  JDK采用“inflation”的技巧：让Java方法在被反射调用时，开头若干次使用native版，等反射调用次数超过阈值时则生成一个专用的MethodAccessor实现类，生成其中的invoke()方法的字节码，以后对该Java方法的反射调用就会使用Java版本。 这项优化是从JDK 1.4开始的。

研究ReflectionFactory.newMethodAccessor()生产MethodAccessor对象的逻辑，一开始(native版)会生产NativeMethodAccessorImpl和DelegatingMethodAccessorImpl两个对象。
DelegatingMethodAccessorImpl的源码如下：

```java
/** Delegates its invocation to another MethodAccessorImpl and can
    change its delegate at run time. */

class DelegatingMethodAccessorImpl extends MethodAccessorImpl {
    private MethodAccessorImpl delegate;

    DelegatingMethodAccessorImpl(MethodAccessorImpl delegate) {
        setDelegate(delegate);
    }

    public Object invoke(Object obj, Object[] args)
        throws IllegalArgumentException, InvocationTargetException
    {
        return delegate.invoke(obj, args);
    }

    void setDelegate(MethodAccessorImpl delegate) {
        this.delegate = delegate;
    }
}
```

它其实是一个中间层，方便在native版与Java版的MethodAccessor之间进行切换。
然后下面就是native版MethodAccessor的Java方面的声明：
sun.reflect.NativeMethodAccessorImpl：

```java
/** Used only for the first few invocations of a Method; afterward,
    switches to bytecode-based implementation */

class NativeMethodAccessorImpl extends MethodAccessorImpl {
    private Method method;
    private DelegatingMethodAccessorImpl parent;
    private int numInvocations;

    NativeMethodAccessorImpl(Method method) {
        this.method = method;
    }

    public Object invoke(Object obj, Object[] args)
        throws IllegalArgumentException, InvocationTargetException
    {
        // We can't inflate methods belonging to vm-anonymous classes because
        // that kind of class can't be referred to by name, hence can't be
        // found from the generated bytecode.
        if (++numInvocations > ReflectionFactory.inflationThreshold()
                && !ReflectUtil.isVMAnonymousClass(method.getDeclaringClass())) {
            MethodAccessorImpl acc = (MethodAccessorImpl)
                new MethodAccessorGenerator().
                    generateMethod(method.getDeclaringClass(),
                                   method.getName(),
                                   method.getParameterTypes(),
                                   method.getReturnType(),
                                   method.getExceptionTypes(),
                                   method.getModifiers());
            parent.setDelegate(acc);
        }

        return invoke0(method, obj, args);
    }

    void setParent(DelegatingMethodAccessorImpl parent) {
        this.parent = parent;
    }

    private static native Object invoke0(Method m, Object obj, Object[] args);
}
```

每次NativeMethodAccessorImpl.invoke()方法被调用时，程序调用计数器都会增加1，看看是否超过阈值；一旦超过，则调用MethodAccessorGenerator.generateMethod()来生成Java版的MethodAccessor的实现类，并且改变DelegatingMethodAccessorImpl所引用的MethodAccessor为Java版。后续经由DelegatingMethodAccessorImpl.invoke()调用到的就是Java版的实现了。
到这里，我们已经追寻到native版的invoke方法在Java一侧声明的最底层 - invoke0了，下面我们将深入到HotSpot JVM中去研究其具体实现。



**invoke流程图：**

![](C:\Users\Administrator\Desktop\学习\java\反射之invoke解析.jpg)

#### 反射的一些注意事项

由于反射会额外消耗一定的系统资源，因此如果不需要动态地创建一个对象，那么就不需要用反射。

另外，反射调用方法时可以忽略权限检查，因此可能会破坏封装性而导致安全问题。

## 八、异常

Throwable 可以用来表示任何可以作为异常抛出的类，分为两种：  **Error**   和 **Exception**。其中 Error 用来表示 JVM 无法处理的错误，Exception 分为两种：

- **受检异常**  ：需要用 try...catch... 语句捕获并进行处理，并且可以从异常中恢复；
- **非受检异常**  ：是程序运行时错误，例如除 0 会引发 Arithmetic Exception，此时程序崩溃并且无法恢复。

 [![img](https://camo.githubusercontent.com/106f71b4556e95b4a8508e82d9b6bec8ccb9e964/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f50506a77502e706e67)](https://camo.githubusercontent.com/106f71b4556e95b4a8508e82d9b6bec8ccb9e964/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f50506a77502e706e67) 

- [Java 入门之异常处理](https://www.tianmaying.com/tutorial/Java-Exception)
- [Java 异常的面试问题及答案 -Part 1](http://www.importnew.com/7383.html)

## 九、泛型

```
public class Box<T> {
    // T stands for "Type"
    private T t;
    public void set(T t) { this.t = t; }
    public T get() { return t; }
}
```

- [Java 泛型详解](http://www.importnew.com/24029.html)
- [10 道 Java 泛型面试题](https://cloud.tencent.com/developer/article/1033693)



## 十、注解

Java 注解是附加在代码中的一些元信息，用于一些工具在编译、运行时进行解析和使用，起到说明、配置的功能。注解不会也不能影响代码的实际逻辑，仅仅起到辅助性的作用。

[注解 Annotation 实现原理与自定义注解例子](https://www.cnblogs.com/acm-bingzi/p/javaAnnotation.html)



## 十一、特性



## Java 各版本的新特性

**New highlights in Java SE 8**

1. Lambda Expressions
2. Pipelines and Streams
3. Date and Time API
4. Default Methods
5. Type Annotations
6. Nashhorn JavaScript Engine
7. Concurrent Accumulators
8. Parallel operations
9. PermGen Error Removed

**New highlights in Java SE 7**

1. Strings in Switch Statement
2. Type Inference for Generic Instance Creation
3. Multiple Exception Handling
4. Support for Dynamic Languages
5. Try with Resources
6. Java nio Package
7. Binary Literals, Underscore in literals
8. Diamond Syntax

- [Difference between Java 1.8 and Java 1.7?](http://www.selfgrowth.com/articles/difference-between-java-18-and-java-17)
- [Java 8 特性](http://www.importnew.com/19345.html)



## Java 与 C++ 的区别

- Java 是纯粹的面向对象语言，所有的对象都继承自 java.lang.Object，C++ 为了兼容 C 即支持面向对象也支持面向过程。
- Java 通过虚拟机从而实现跨平台特性，但是 C++ 依赖于特定的平台。
- Java 没有指针，它的引用可以理解为安全指针，而 C++ 具有和 C 一样的指针。
- Java 支持自动垃圾回收，而 C++ 需要手动回收。
- Java 不支持多重继承，只能通过实现多个接口来达到相同目的，而 C++ 支持多重继承。
- Java 不支持操作符重载，虽然可以对两个 String 对象执行加法运算，但是这是语言内置支持的操作，不属于操作符重载，而 C++ 可以。
- Java 的 goto 是保留字，但是不可用，C++ 可以使用 goto。

[What are the main differences between Java and C++?](http://cs-fundamentals.com/tech-interview/java/differences-between-java-and-cpp.php)



## JRE or JDK

- JRE：Java Runtime Environment，Java 运行环境的简称，为 Java 的运行提供了所需的环境。它是一个 JVM 程序，主要包括了 JVM 的标准实现和一些 Java 基本类库。
- JDK：Java Development Kit，Java 开发工具包，提供了 Java 的开发及运行环境。JDK 是 Java 开发的核心，集成了 JRE 以及一些其它的工具，比如编译 Java 源码的编译器 javac 等。