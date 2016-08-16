---
layout: post
title: Guava Primitives
categories: [blog]
tags: [Guava]
description: 
---

## Guava Primitives

### 基本类型对应关系

jdk原生类型|com.google.common.primitives.*
byte	|Bytes, SignedBytes, UnsignedBytes
short	|Shorts
int	        |Ints,UnsignedInteger, UnsignedInts
long	|Longs, UnsignedLong, UnsignedLongs
float	|Floats
double	|Doubles
char	|Chars
boolean	|Booleans

### 工具

Arrays.asList|把数组转为相应包装类的List
Collection.toArray()|把集合拷贝为数组，和collection.toArray()一样线程安全
Iterables.concat|串联多个原生类型数组
Collection.contains|判断原生类型数组是否包含给定值
List.indexOf|给定值在数组中首次出现处的索引，若不包含此值返回-1
List.lastIndexOf|给定值在数组最后出现的索引，若不包含此值返回-1
Collections.min|数组中最小的值
Collections.max|数组中最大的值
Joiner.on(separator).join|把数组用给定分隔符连接为字符串
Ordering.natural().lexicographical()|按字典序比较原生类型数组的Comparator