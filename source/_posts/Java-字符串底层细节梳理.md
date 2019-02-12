---
title: Java 字符串底层细节梳理
date: 2019-01-27 14:15:27
tags: [Java]
categories: [Java]
toc: true
---

## 类图

![](https://wx2.sinaimg.cn/mw1024/7c35df9bly1fzonkgeuwkj20kd08y0sr.jpg)

### 成员变量

```java
    /** 
     * The value is used for character storage. 
     * 存储character 的字节数组， final
     */
    private final char value[];

    /** 
     * Cache the hash code for the string 
     * 缓存字符串的Hash值， 提升代码运行效率
     */
    private int hash; // Default to 0
```

### Why immutable

- **安全（Security）**: 网络连接和数据库链接都使用String作为参数， 如url， username， password， 如果String是mutable的， 这些参数就很容易被篡改； 

- **类加载（Class loading）**: Java中类加载过程中使用String作为参数， 如果String 是mutable的， 就可能导致类加载过程出错。 

- **缓存/String Pool()**: 缓存主要涉及到两个方面：
  1. 一方面是String内部char[]数组本身的StringPool缓存； 
  2. hashCode缓存， String本身的immutable， 能够保证其hashCode不变， 这可以提升以String为key的HashMap查询效率;

  ![](https://wx1.sinaimg.cn/mw690/7c35df9bly1fzptb34pjlj20ic0b4jrs.jpg) 

- **同步（Synchronization）**: String immutable 使得多个线程之间可以安全的共享同一个字符串；  

### Really immutable?

```java
    String string = new String("abc");
    System.out.println(string);

    Field valueField = string.getClass().getDeclaredField("value");
    valueField.setAccessible(true);

    char[] abcd = new char[] {'a', 'b', 'c', 'd'};
    valueField.set(string, abcd);

    System.out.println(string);
```

上面的代码标明： 通过反射我们还是能够改变String内部char数组的value引用指向， 所以String还是可以改变的。 

## StringBuilder

## StringBuffer

## 参考引用

- [1] [Why is String immutable in Java?](https://stackoverflow.com/questions/22397861/why-is-string-immutable-in-java)
- [2] [Why String is Immutable in Java](https://dzone.com/articles/why-string-immutable-java)

