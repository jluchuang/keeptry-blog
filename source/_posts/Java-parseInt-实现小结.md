---
title: Java parseInt 实现小结
date: 2019-01-27 13:54:12
toc: true
tags: [Java, Algorithm]
categories: 
  - [Java]
  - [Algorithm]
description: 
---

按 JDK 1.8的注释描述， Java Integer.parseInt 方法主要用来实现将输入字符串参数按照给定的进制转换成10进制int型整数返回。 

jdk注释如下：  

```java
 /**
     * Parses the string argument as a signed integer in the radix
     * specified by the second argument. The characters in the string
     * must all be digits of the specified radix (as determined by
     * whether {@link java.lang.Character#digit(char, int)} returns a
     * nonnegative value), except that the first character may be an
     * ASCII minus sign {@code '-'} ({@code '\u005Cu002D'}) to
     * indicate a negative value or an ASCII plus sign {@code '+'}
     * ({@code '\u005Cu002B'}) to indicate a positive value. The
     * resulting integer value is returned.
     *
     * <p>An exception of type {@code NumberFormatException} is
     * thrown if any of the following situations occurs:
     * <ul>
     * <li>The first argument is {@code null} or is a string of
     * length zero.
     *
     * <li>The radix is either smaller than
     * {@link java.lang.Character#MIN_RADIX} or
     * larger than {@link java.lang.Character#MAX_RADIX}.
     *
     * <li>Any character of the string is not a digit of the specified
     * radix, except that the first character may be a minus sign
     * {@code '-'} ({@code '\u005Cu002D'}) or plus sign
     * {@code '+'} ({@code '\u005Cu002B'}) provided that the
     * string is longer than length 1.
     *
     * <li>The value represented by the string is not a value of type
     * {@code int}.
     * </ul>
 */
```

## 边界条件

### **空串/null参数**

### **进制radix超界**

支持最小进制 2进制， 最大 36进制， 因为最多只有26个字母; 

```java
/**
     * The minimum radix available for conversion to and from strings.
     * The constant value of this field is the smallest value permitted
     * for the radix argument in radix-conversion methods such as the
     * {@code digit} method, the {@code forDigit} method, and the
     * {@code toString} method of class {@code Integer}.
     *
     * @see     Character#digit(char, int)
     * @see     Character#forDigit(int, int)
     * @see     Integer#toString(int, int)
     * @see     Integer#valueOf(String)
     */
    public static final int MIN_RADIX = 2;

    /**
     * The maximum radix available for conversion to and from strings.
     * The constant value of this field is the largest value permitted
     * for the radix argument in radix-conversion methods such as the
     * {@code digit} method, the {@code forDigit} method, and the
     * {@code toString} method of class {@code Integer}.
     *
     * @see     Character#digit(char, int)
     * @see     Character#forDigit(int, int)
     * @see     Integer#toString(int, int)
     * @see     Integer#valueOf(String)
     */
    public static final int MAX_RADIX = 36;
```

### **整数上溢/下溢** 

使用”负数相减“的方式来代替累加操作; 

```java
    /**
     * A constant holding the minimum value an {@code int} can
     * have, -2<sup>31</sup>.
     */
    @Native public static final int   MIN_VALUE = 0x80000000;

    /**
     * A constant holding the maximum value an {@code int} can
     * have, 2<sup>31</sup>-1.
     */
    @Native public static final int   MAX_VALUE = 0x7fffffff;
```


## 实现源码 

```java
    /*
     * @param      s   the {@code String} containing the integer
     *                  representation to be parsed
     * @param      radix   the radix to be used while parsing {@code s}.
     * @return     the integer represented by the string argument in the
     *             specified radix.
     * @exception  NumberFormatException if the {@code String}
     *             does not contain a parsable {@code int}.
     */
    public static int parseInt(String s, int radix)
                throws NumberFormatException
    {
        /*
         * WARNING: This method may be invoked early during VM initialization
         * before IntegerCache is initialized. Care must be taken to not use
         * the valueOf method.
         */
        
        // null入参判断
        if (s == null) {
            throw new NumberFormatException("null");
        }

        // 最小支持二进制入参
        if (radix < Character.MIN_RADIX) {
            throw new NumberFormatException("radix " + radix +
                                            " less than Character.MIN_RADIX");
        }
        // 最大支持36进制
        if (radix > Character.MAX_RADIX) {
            throw new NumberFormatException("radix " + radix +
                                            " greater than Character.MAX_RADIX");
        }

        int result = 0;                     // 最终返回结果
        boolean negative = false;           // 符号标识
        int i = 0, len = s.length();
        int limit = -Integer.MAX_VALUE;     // 正数上界
        int multmin;
        int digit;

        if (len > 0) {
            char firstChar = s.charAt(0);
            // 处理符号位
            if (firstChar < '0') { // Possible leading "+" or "-"
                if (firstChar == '-') {
                    negative = true;
                    limit = Integer.MIN_VALUE;  // 处理负数下界
                } else if (firstChar != '+')
                    throw NumberFormatException.forInputString(s);

                if (len == 1) // Cannot have lone "+" or "-"
                    throw NumberFormatException.forInputString(s);
                i++;
            }
            // 最后一次乘法运算防止出现溢出
            // 因为正负数limit不同， 这里multmin的值可能不同
            multmin = limit / radix;
            while (i < len) {
                // Accumulating negatively avoids surprises near MAX_VALUE
                // while循环中， 采用负数减法的形式累加， 因为负数的下界2^31比正数上界(2^31 - 1)大1 
                // 获取对应入参高位第i位的数值(返回对应的radix进制数)
                digit = Character.digit(s.charAt(i++),radix);
                // 非法的进制数值 
                if (digit < 0) {
                    throw NumberFormatException.forInputString(s);
                }
                // 判断乘法计算的溢出
                if (result < multmin) {
                    throw NumberFormatException.forInputString(s);
                }
                // 进行乘法运算
                result *= radix;
                // 判断加法运算出现的溢出
                if (result < limit + digit) {
                    throw NumberFormatException.forInputString(s);
                }
                result -= digit;
            }
        } else {
            // 空串判断
            throw NumberFormatException.forInputString(s);
        }
        return negative ? result : -result;
    }
```

## 总结

Java parseInt方法实现是比较优雅的， 充分考虑了计算过程中可能出现的边界条件， 并通过负数相减的方式有效的避免了转换过程中出现溢出的情况。 
