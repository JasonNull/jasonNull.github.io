---
title: instanceof
date: 2018-06-23 16:19:33
tags:
- java
- 基础
- 关键字
categories:
- java
- 基础
toc: true
---
# instanceof
javac 有一个INSTANCEOF指令
jvm的实现是，会对每一个对象，有一个主要超类型链（实现为数组，深度为7以内的超类），可能会有一个次要超类型链表（实现为数组，接口，以及深度超过8的超类)
instanceof会执行一下这两个方法，查找主要超类型和次要超类型链
下面是hotspot的实现
```c++
S.is_subtype_of(T) := {
  int off = T.offset;
  if (S == T) return true;
  if (T == S[off]) return true;
  if (off != &cache) return false;
  if ( S.scan_secondary_subtype_array(T) ) {
    S.cache = T;
    return true;
  }
  return false;
}
```
