---
title: 解读不可变的String
date: 2018-03-31 09:01:21
tags: Java
---

### 前言

在JDK API的对String的描述中，有以下对String的介绍：

> String 类代表字符串。Java 程序中的所有字符串字面值（如 "abc" ）都作为此类的实例实现。</br>
> 字符串是常量；它们的值在创建之后不能更改。字符串缓冲区支持可变的字符串。因为 String 对象是不可变的，所以可以共享。

<!-- more -->

在这里指出了String对象时不可变的，那么下面我们来具体分析为什么java中的String是不可变的。

### 不可变对象 ###
不可变对象是指一个对象的状态在对象被创建之后就不再变化。不可改变的意思就是说：不能改变对象内的成员变量，包括基本数据类型的值不能改变，引用类型的变量不能指向其他的对象，引用类型指向的对象的状态也不能改变。

那么会有人说，以下的代码不就改变了String吗：
```Java
public static void main(String[] args) {
    String s = "ABCDEF";
    System.out.println("s = " + s);
    
    s = "123456";
    System.out.println("s = " + s);
}
```
打印的结果：</br>
s = ABCDEF</br>
s = 123456

* 你看，从结果上看，确实是如此，s的值改变了，这不是和我们说的不可变对象String完全不搭边。
* 其实，这里存在一个误区，s呢 ，仅仅是一个String对象的引用，并不是对象本身。对象在内存中是一块内存区，而s只是一个引用，它指向了一个具体的对象。
* 在创建String对象的时候，s指向的是内存中的"ABDCEF",当执行语句`s = "123456"`后，其实又创建了一个新的对象"123456",而s重新指向了这个新的对象，同时原来的"ABCDEF"并没有发生改变，仍保存在内存中。

> 在这里，对象引用和C语言中的指针其实很像，它们都是存放的对象在内存中的地址值，只是在java中的引用丧失了部分灵活性，比如就不能像C中的指针那样进行运算。

那么又会有朋友说啦，String类中有修改自身的方法又是怎样呢，比如不是有一个replaced(替换)方法么
```java
public static void main(String[] args) {
    String s = "ABCDEF";
    System.out.println("s = " + s);
    
    s.replace("A", "a");
    System.out.println("s = " + s);
}
```
打印结果：</br>
s = ABCDEF</br>
s = ABCDEF</br>

结果是不是没有改变，是否出乎你的意料呢，现在改一下代码：
```java
public static void main(String[] args) {
    String s = "ABCDEF";
    System.out.println("s = " + s);
    
    s = s.replace("A", "a");
    System.out.println("s = " + s);
}
```
打印结果：</br>
s = ABCDEF</br>
s = aBCDEF</br>

这时候你会发现，结果确实是替换了，但是并不是说String对象发生了改变，而是执行语句`s = s.replace("A", "a")`将引用s指向了一个新的String对象（"aBCDEF"），原来的"ABCDEF"还在内存并没有被改变。

### 如何实现不可变 ###
要了解String的不可变性，首先看一下String类中都有哪些成员变量，
```java
/** The value is used for character storage. */
private final char value[];

/** Cache the hash code for the string */
private int hash; // Default to 0
```
* 从注释中，可以看出value变量是String用来存放字符的，还有一个hash成员变量是该String对象的哈希值缓存。
* value，hash这两个成员变量都是private final修饰的，而String类中并没有提供set/get等公共方法来修改这些值，所以String类的外部无法修改String，同时final保证了在String内部，只要值初始化了，也不能被改变。
* 所以认为String对象是不可变的。
* 而在java中，数据也是对象，那么value也仅仅是一个引用，指向真正的数组对象，这将在下面的文章中用到。

### 不可变对象的好处 ###
1. 不可变对象更容易构造，测试与使用；
2. 真正不可变对象都是线程安全的；
3. 不可变对象的使用没有副作用（没有保护性拷贝）；
4. 对象变化的问题得到了避免；
5. 不可变对象的失败都是原子性的；
6. 不可变对象更容易缓存，且可以避免null引用；
7. 不可变对象可以避免时间上的耦合；


 **String，StringBuffer，StringBuilder，都是final类，不允许被继承，在本质上都是字符数组**，不同的是，String的长度是不可变的而后两者长度可变，在进行连接操作时，String每次返回一个新的String实例，而StringBuffer和StringBuilder的append方法直接返回this，所以当进行大量的字符串连接操作时，不推荐使用String，因为它会产生大量的中间String对象。

StringBuffer和StringBuilder的一个区别是，StringBuffer在append方法前增加了一个synchronized修饰符，以起到同步的作用，为此也降低了执行效率；若要在toString方法中使用循环，使用StringBuilder

### 那么，真的不可变么 ###
现在我们已经知道了String的成员变量是private final 的，也就是初始化之后不可改变的。同时也提到value这个成员变量其实也是一个引用，指向真正的数组内存地址，不能改变它的引用指向，我们能不能直接改变内存数组中的数据呢，那么就需要获取到value，而value是私有的，我们怎么获取呢？对！就是用反射。
```Java
public static void reflectString() throws Exception{
    
    String s = "ABCDEF";
    System.out.println("s = " + s);
    
    Field valueField = s.getClass().getDeclaredField("value");
    valueField.setAccessible(true);
    
    char[] value = (char[]) valueField.get(s);
    value[0] = 'a';
    value[2] = 'c';
    value[4] = 'e';
    
    System.out.println("s = " + s);
}
```
打印结果：</br>
s = ABCDEF</br>
s = aBcDeF</br>
结果显而易见！