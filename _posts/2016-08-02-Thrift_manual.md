---
layout: post
title: Thrift manual
categories: [blog]
tags: [SOA]
description: 
---

# Thrift manual

[TOC]

## Install

`brew install thrift`

## Grammer

### namespace
As same as "package" in Java.
`namespace java resource.thrift.sdk.hermes`

结合mv，管理路径。


### Primary

- byte: 有符号字节
- i16: 16位有符号整数
- i32: 32位有符号整数
- i64: 64位有符号整数
- double: 64位浮点数
- string: 字符串

### Collection

Element in collection can be any types except service.

- list<T>
- set<T>
- map<K, V>

### Enum
```
enum THermesTemplateVerifyStatus {
    PENDING = 0
    SUCCESS = 1
    FAIL = 2
}
```
### Struct
```
struct THermesUserReply {
    1: required i64 id
    2: required string phone_number
    3: required Timestamp reply_time
    4: required string message
}
```
会生成一个同名class(Java Bean)
bool/byte/i16/i32/i64/double/enum类型的处理逻辑
[required属性字段]转为原生类型：boolean/byte/short/int/long/double/int
[optional属性字段]转为装箱类型：Boolean/Byte/Short/Integer/Long/Double/Integer
[default属性字段]转为装箱类型：Boolean/Byte/Short/Integer/Long/Double/Integer
如服务端实现中有特殊逻辑，可以在和服务方确认逻辑后手动修改类型
Thrift中每个字段都会生成一个private字段，一个public getter，一个public setter
生成的每个字段都会打上@Index(index)注解
字段不支持默认值

### Service

```
service HermesService {

    bool ping()
        throws (1: HermesUserException user_exception,
                2: HermesSystemException system_exception,
                3: HermesUnknownException unknown_exception),

```

### Exception

```
exception HermesUnknownException {
    1: required HermesErrorCode error_code,
    2: required string error_name,
    3: required string message,
}
```

## Generate

`thrift -gen java hermes.thrift`

## Tutorial

[tutorial](http://thrift-tutorial.readthedocs.io/en/latest/thrift-file.html)

[基础使用教程](http://my.oschina.net/jack230230/blog/66041)

[基础使用教2](http://www.jianshu.com/p/0f4113d6ec4b)

[Java版的各种Thrift server实现的比较](http://www.codelast.com/%E5%8E%9F%E5%88%9B%EF%BC%88%E7%BF%BB%E8%AF%91%EF%BC%89java%E7%89%88%E7%9A%84%E5%90%84%E7%A7%8Dthrift-server%E5%AE%9E%E7%8E%B0%E7%9A%84%E6%AF%94%E8%BE%83/)

