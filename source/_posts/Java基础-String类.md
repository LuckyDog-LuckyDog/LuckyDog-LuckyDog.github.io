---
title: Java基础--String类
date: 2020-03-07 23:09:55
tags: [String]
categories: [Java]
---

回顾String类 , 温故而知新

<!--more-->

# String类

- String是一个final类 ,代表不可变性 , 不可以被继承
- 字符串是常量, 用双引号引起表示,值被创建之后是不能更改的
- String 的底层是一个字符数组value[] 
- 实现了Serializable(支持字符串序列化), Comparable(支持字符串比较) , CharSequence(代表是一个字符串)接口

```java
//String的底层
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
```

## 1 代码验证

```java
        String a = "xiaoliu";//字面量 , 存于常量池中
        String b = "xiaoliu";
        System.out.println(a==b);//true ,常量池不会存储同一个字符
        a="xiaolin";
        System.out.println(a==b);//false ,重新赋值时 ,重新指定内存区域
        String c ="xiaoliu";
        c+="xiaolin";
        System.out.println(c);//xiaoliuxiaolin 连接操作时,重新指向另外的内存区域
        String d = "Xiaolin";
        String replace = a.replace("x", "X");
        System.out.println(replace);//Xiaolin
        System.out.println(replace==a);//false , 代替操作时,重新指定内存区域
        System.out.println(replace==d);//false
```

![3jqCUx.png](https://s2.ax1x.com/2020/03/07/3jqCUx.png)



## 2 String的创建

三种创建String对象方式 : 

1. ""字面量赋值
2. new String() 构造方法赋值
3. set方法赋值

```java
    String S1 = new String();//本质this.value = new char[0]
    String S2 = new String(String original)//this.value = original.value
    String S3 = new String(char[] a) // this.value = Arrays.copyOf(value ,value.length);
    String S4 = new String (char[] a ,int startIndex , int count)
```

**注意 :** **new String("xiaoliu")这个过程中创建了两个对象 :**

**1)new结构 ;**

**2) 底层chart[] ,字符串常量池**

```java
        String a1 = "xiaoliu";//字面量 , 存于方法区的常量池空间
        String a2 = "xiaoliu";
        String b1 = new String("xiaoliu");//new 为构造函数 , 存于堆内存空间
        String b2 = new String("xiaoliu");
        System.out.println(a1 == a2);//true
        System.out.println(a1 == b1);//false
        System.out.println(a1 == b2);//false
        System.out.println(b1 == b2);//false
        Person person = new Person("xiaoliu", 18);
        Person person1 = new Person("xiaoliu", 18);
        System.out.println(person.name == person1.name);//true

        String s1 = "xiaoliu";
        String s2 = "xiaolin";
	    final String s8 = "xiaolin";
        String s3 = "xiaoliu" + "xiaolin";
        String s4 = "xiaoliuxiaolin";
		System.out.println(s8 == s4); //true ,使用了final 意为不可变
        String s5 = "xiaoliu" + s2;
        String s6 = s1 + "xiaolin";
        String s7 = s1 + s2;
        System.out.println(s3 == s4);//true , 连接操作 , 存于常量池空间
        System.out.println(s3 == s5);//false , 只要过程有变量参与 , 那么存于堆内存空间
        System.out.println(s3 == s6);//false
        System.out.println(s3 == s7);//false
        System.out.println(s5 == s6);//false
        System.out.println(s5 == s7);//false
        System.out.println(s6 == s7);//false
        String s8 = s5.intern();
        System.out.println(s8==s4);//true , 注意jdk8中如果常量池中有就会返回常量池中的对象 , jdk8之前都是直接创建新的对象,放在这里返回的是fasle
```

## 3 String的常见用法

```java
int leng() : 返回字符串的长度:return value.length
char charAt(int index): 返回某索引处return value[index]
boolean isEmpty():判断是否为空字符  :return value.length==0;
String toLowerCase() : 转化为小写
String toUpperCase() : 转化为大写
String Trim() : 返回字符串副本,忽略前部空白和尾部空白
boolean equal(Object obj) : 比较字符串的内容是否相同
boolean equalIgnoreCase(String anotherString):与equals类似 , 忽略大小写
String concat (String str) :将指定字符串连接到此字符串的结尾 , 等价与"+"
int compareTo(String anotherString) : 比较两个字符串的大小
String substring(int beginIndex):返回一个新的字符串 , 以beginIndex开始截取
String substring(int beginIndex,int endIndex)返回一个新的字符串 ,从begin到end截取,不包含第一个字符
boolean endsWith(String Suffix):是否以指定的后缀结束
boolean startWith(String prefix):是否以指定给的前缀结束
boolean startWith(String prefix,String Suffix)
boolean contains(ChartSequence s) :
int indexOf(String str):第一次出现Str的索引
int indexOf(String str , int fromIndex):
int lastIndexOf(String str):返回指定子字符串在此字符串中最右边出现处的索引
int lastIndexOf(String str ,int fromIndex):
String replace(char oldChar . char newchar):
String replace(charSequence target . charSequence replacement):
String repaceAll(String regex , Sreing replacement)
String replaceFirst(String regex , String replacement)
  boolean matches(String regex) : 是否正则表达式匹配
  String split(String regex):
  String split(String regex , int limit): 根据匹配的正则表达式来拆分次字符串
//index未找到都会放回-1 , indexOf==lastIndexOf的情况 : 1.只有一个字符串 2. 都找不到
```

## 4 String与基本数据类型的转化

- 字符串 - > 基本数据类型 包装类

parseInt(String s):将数字字符串转化为整型 ,Byte ,Short,Long,Float,Double 都可以装化为基本相应的基本数据类型

- 基本数据类型  , 包装类 - > 字符串

String.ValueOf() 可将()里面的东西转化为字符串

## 5 String与字符数组的转化

String --> chart[] , 调用toChartArray()方法
chart[] --> String , 调用String的构造 , new String(chart[ ]) 

## 6 String与字节数组的转化

String --> byte[] , 调用getBytes()方法 , 里面可以指定编码
 byte[] --> String ,  调用String的构造 , new String( byte[ ]) , 可以指定编码

## 7 String - StringBuffer - StringBuider

String : 不可变字符串序列 ,底层是chart[]存储
StringBuffer  : 可变字符串序列, 线程安全 ,方法都加了 synchronized方法 ,效率低下 ,底层是chart[]存储
StringBuider : 可变字符串序列 , 线程不安全 , 效率高 ,底层是chart[]存储

### (1) 源码分析

```java
//String
String str = new String();//char[] value = new char[0];
String str1 = new String("abc")//char[] value= new char[]{'a','b','c'}
  //StringBuffer
StringBuffer sb = new StringBuffer();//char[] value=new char[16];底层创建一个长度为16的字符数组
System.out.println(sb.length);//0 , 底层返回的是count 个数
sb.append('a') //value[0] ='a';
sb.append('b')//value[1]='b';
 String sb1 = new StringBuffer("xiaoliu") ; //char[] value = new char["xiaoliu".length+16];

```

问题  :当可字符串序列的底层数组不够时 , 底层会判断(当前长度+新增长度)=max的和是否大于16 , 如果大于 ,就新建数据 ,然后将值copy过来 , 见下面源码分析;

```java
    private void ensureCapacityInternal(int minimumCapacity) {
        // overflow-conscious code
        if (minimumCapacity - value.length > 0) {
          //如果新增的长度大于数组的原始长度16 , 就进行Copy
            value = Arrays.copyOf(value,
                    newCapacity(minimumCapacity));
        }
    }
    private int newCapacity(int minCapacity) {
        // overflow-conscious code
        int newCapacity = (value.length << 1) + 2;//将16的字节数组扩容为原来的2倍+2
        if (newCapacity - minCapacity < 0) {//如果两倍还不够 ,那么直接变为你的长度
            newCapacity = minCapacity;
        }
        return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
            ? hugeCapacity(minCapacity)
            : newCapacity;
    }
//所以在使用的时候能够用指定StringBuffer的长度 , 就是避免扩容的情况 ,提高效率
//StringBuffer(int capacity) 或 StringBuider(int capacity)
```

### (2) StringBuffer常用方法

![3jbN4K.png](https://s2.ax1x.com/2020/03/07/3jbN4K.png)

### (3) 三者效率的比较

StringBuilder > StringBuffer > String







