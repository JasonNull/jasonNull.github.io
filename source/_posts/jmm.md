---
title: jmm
date: 2018-06-23 15:56:13
tags: 
- java
- jvm
- jmm
categories:
- java
- jvm
toc: true
---
# 内存区域
1. 程序计数器
2. java方法栈
3. 本地方法栈
4. 程序计数器
5. 方法区

# 方法区
1. 类信息
2. 常量池
3. 静态变量
4. javac编译产生的符号引用，字面量，jit编译优化后的代码
5. 等等

<!-- more -->

# 常量池
1. class文件中的常量池，字面量（文本字符串等），符号引用（接口，类，方法，字段名称）
2. 运行时常量池，编译期（javac编译）生成的各种字面量和符号引用
3. 全局字符串常量池，StringTable

```java
String s = new String("123");
String s2 = s.intern()
// "123"被存储在第一部分常量池中
// s被存储在堆内存中
// s2则被存储在StringTable中了
```

# 永久代变动
为什么移除永久代
1. 字符串存储在永久代中，容易溢出
2. 类相关信息比较难确定大小，容易溢出
3. 永久代为gc带来了不必要的复杂度，回收效率偏低

## jdk6
1. 上面的常量池
2. 类信息（动态的Class对象，Method，Field等对象，**jit编译优化过后的机器码存储在native内存**）
3. 静态变量

## jdk7
1. 符号引用移动到native heap（8中的元数据区）
2. 字面量移动到java heap（包含StringTable）
3. 静态变量移到java heap

## jdk8
彻底移除永久代，取而代之产生了MetaSpace

## jmm
### 主内存和工作内存
主内存，对标内存，工作内存对标cpu缓存（或者寄存器）。
### 内存操作指令
1. read，从主内存传输到工作内存
2. load，从read回来的数据装载到工作内存中的副本
3. write，将传输回来的值，写入到主内存
4. store，从工作内存传输回主内存，供write操作使用
5. lock，对变量加锁
6. unlock，对变量解锁
7. use，使用变量
8. assign，对变量复制