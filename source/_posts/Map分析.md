---
title: Map分析
date: 2018-06-25 20:01:36
tags:
- java
- 集合
- 源码
categories:
- java
- 集合
toc: true
---
# HashMap
## 成员属性
1. Node<K,V>[]table，存储数据的数组
2. Set<Map.Entry<K,V>>entrySet，遍历时使用的entrySet
3. size，当前大小
4. modCount，被非幂等操作改变的次数，entrySet中的ConcurrentModificationException
5. threshold，下次resize时候可能需要满足的大小
6. loadFactor，负载因子，size > (threshold = capacity * loadFactor)的时候，可能会resize，存在的作用是用于开发人员手动控制resize时具体的数组元素的多少。
<!-- more -->

## 静态属性
1. DEFAULT_INITIAL_CAPACITY，初始大小，16
2. MAXIMUM_CAPACITY，最大大小，2的30次方
3. DEFAULT_LOAD_FACTOR，初始负载因子，0.75
4. TREEIFY_THRESHOLD，treeify时候bin的最小大小，8
5. UNTREEIFY_THRESHOLD，untreeify时bin内元素的最小大小，6
6. MIN_TREEIFY_CAPACITY，最小treeify的容量，64，大于64才会treeify

## 主要方法
### hash
虽然HashMap的bucket数量最大为Integer.MAX_VALUE，但是绝大部分的Map的bucket数量都是较小的，hash码也只用到低n位。
```java
static final int hash(Object key) {
    int h;
    // 混合原始哈希码的高位和低位，以此来加大低位的随机性
    // 主要还是为了避免用户的hashCode算法有问题，低位散列不均衡
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
### get
1. 根据key的hash，计算出bucket的位置
2. 如果bucket为空，返回空
3. 如果是链表，遍历链表，其中使用equals比较
4. 如果是红黑树的话，遍历整颗树，其中使用hash比较或者使用Comparable的compareTo方法作比较

### put
使用尾插法，因为put的过程中需要遍历整个链表，查看是否存在相同的key
1. 如果table为空，进行resize()
2. 根据hash计算bucket的index
3. 取得bucket后，如果是空，构建链表
4. 如果是链表节点，（如果需要树化，树化，走5），插入至链表头部
5. 如果是树节点，插入树节点，并且进行平衡
6. 最后需要resize，进行resize()

### resize
resize之后大小扩大两倍
什么时候会进行resize？进一步观察下来，两种情况下会进行resize。
1. 某个bin内的元素数量 > TREEIFY_THRESHOLD（8）, 但是总体的Capacity小于MIN_TREEIFY_CAPACITY（64）
2. put操作过后，size > (threshold = capacity * loadFactor) 的时候也会。

resize代码分析：
下面这段代码很精妙，巧妙地利用了一个点。
假设元素e，oldCapacity，oldPos
```java
// bucket内链表拆分落在，oldPos以及oldPos + oldCapacity两个新的bucket内
e.newPos = e.hash & oldCapacity == 1 ? oldPos + oldCapacity: oldPos;
// 下面是jdk代码，将链表做了精妙的拆解，拆成两段
// 两个好处
// 1. 不会倒置旧链表
// 2. 效率更高
do {
    next = e.next;
    if ((e.hash & oldCap) == 0) {
        if (loTail == null)
            loHead = e;
        else
            loTail.next = e;
        loTail = e;
    }
    else {
        if (hiTail == null)
            hiHead = e;
        else
            hiTail.next = e;
        hiTail = e;
    }
} while ((e = next) != null);
if (loTail != null) {
    loTail.next = null;
    newTab[j] = loHead;
}
if (hiTail != null) {
    hiTail.next = null;
    newTab[j + oldCap] = hiHead;
}
```

### entrySet, keySet
#### 原理
两个都是针对table属性进行foreach循环遍历，但是keySet只能取出key，entrySet能取出Entry。两者都使用了modCount用来保证fail-fast，ConcurrentModificationException。
#### 用法
只需要遍历key的话还是推荐使用keySet()，但是不推荐下面这种用法。

# ConcurrentHashMap
## jdk7
1. 分段锁，内部实现为Segment，继承了ReentrantLock
2. 默认16段，最大为65536，Segment内部有HashEntry数组，并且采取拉链法处理hash冲突
3. 扩容的时候针对于Segment内部的HashEntry数组来扩容。
4. 需要注意的是jdk中有concurrencyLevel（代表Segment数组的大小），在8中已经没什么用处了，只能决定初始容量大小

## 静态属性
1. MAXIMUM_CAPACITY，table的最大大小，1<<30
2. DEFAULT_CAPACITY，table的大小，16
3. MAX_ARRAY_SIZE，元素最大个数，Integer.MAX_VALUE-8
4. DEFAULT_CONCURRENCY_LEVEL，向jdk7兼容，16
5. LOAD_FACTOR，负载因子，0.75f
6. TREEIFY_THRESHOLD，进行树化的元素数量，8
7. UNTREEIFY_THRESHOLD，untreeify时bin内元素的最小大小，6
8. MIN_TREEIFY_CAPACITY，进行treeify时需要的最要容量要求，64
9.  MIN_TRANSFER_STRIDE，多线程扩容时候的步长
10. RESIZE_STAMP_BITS，
11. MAX_RESIZERS，帮助resize的最大线程数
12. RESIZE_STAMP_SHIFT，
13. MOVED（-1），TREEBIN（-2），RESERVED（-3），hash的常量值，分别代表当前node是ForwardingNode，TreeNode，ReservationNode的hash值。
    - ForwardingNode代表当前bin已经被resize操作过，并且需要当前线程一起帮助resize
    - TreeNode代表当前
    - Node就是普通的链表节点
    - ReservationNode代表当前节点已经被占据（占位符的作用一样），用于后续的mappingFunction操作结束之后再把值塞入进去。

## 成员属性
1. table，存储元素的数组
2. nextTable，也是存储元素的数组，只不过该数组在resizing的时候才会被用到。
3. baseCount，容器中存储元素的数量值（基本值）
4. sizeCtl，代表当前容器中正在进行的关于容量的操作，-1，代表正在被初始化，≤-1，代表正在被（-sizeCtl-1)个线程扩容，≥0代表0.75*capacity
5. transferIndex，代表已经或者正在被扩容处理的节点，未被处理的bin的范围为0~transferIndex - 1
6. cellsBusy，代表当前正在初始化某个CounterCell，用于控制多个线程不能同时初始化一个CounterCell
7. counterCells，计数器，ConcurrentHashMap的真正size是baseCount+CounterCell数组中的计数。具体看LongAdder分析
8. keySet，此三个属性为迭代器视图
9. values
10. entrySet

## 主要方法
### resize
1. 需要判断是否是第一个线程，如果是第一个线程还需要进行新建数组操作
2. 假设当前容量为64，需要扩容到128，一共三个线程扩容
3. 假设stride是16，capacity是64，两个线程正在参与扩容操作
4. 第一个线程处理(48,64]这个范围的元素
5. 第二个线程处理(32,48]
6. 谁先处理完手里的元素，就将transferIndex减去stride(32-16),就会继续处理(16,32]这个范围一样
7. 其他线程进来之后，也会减少transferIndex   
8. 链表的拆分或者树的拆分都是新建了全新的链表或者是树。
9. 链表会被倒置。树则不会，会再次平衡 

### putVal
1. 如果未初始化数组，cas操作sizeCtl为-1，初始化数组
2. 如果未初始化bin（bucket），cas数组元素初始化
3. 如果当前bin的首元素是ForwardingNode，帮助进行resize
4. 在数组已经初始化，当前bin未被扩容的情况下，对当前bin的头结点，加锁，进行链表或者红黑树的插入
5. 插入完成之后判断是否需要进行树化，或者进行resize

<img src="/images/putVal1.png" width="60%" style="margin-left:0;">
<img src="/images/putVal2.png" width="60%" style="margin-left:0;">

### get
<img src="/images/get1.png" width="60%" style="margin-left:0;">
<img src="/images/get2.png" width="60%" style="margin-left:0;">

# LinkedHashMap
数据倾斜：由于hash算法设计不合理，或者原本输入传不合理，导致hash结果集中度太高
## 成员属性
1. `LinkedHashMap.Entry<K,V> head`，表示链表的头结点
2. `LinkedHashMap.Entry<K,V> tail`，表示链表的尾节点
3. `boolean accessOrder`，表示当前插入节点时候时候需要维护有序
<!-- more -->

## 主要方法
```java
// 将当前节点加入到链表的最后去
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}

// 将某个节点替换成另外一个
private void transferLinks(LinkedHashMap.Entry<K,V> src,
                            LinkedHashMap.Entry<K,V> dst) {
    LinkedHashMap.Entry<K,V> b = dst.before = src.before;
    LinkedHashMap.Entry<K,V> a = dst.after = src.after;
    if (b == null)
        head = dst;
    else
        b.after = dst;
    if (a == null)
        tail = dst;
    else
        a.before = dst;
}

// 新建节点，可以看出使用的是LinkHashMap.Entry
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}

// 替换节点
Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
    LinkedHashMap.Entry<K,V> q = (LinkedHashMap.Entry<K,V>)p;
    LinkedHashMap.Entry<K,V> t =
        new LinkedHashMap.Entry<K,V>(q.hash, q.key, q.value, next);
    transferLinks(q, t);
    return t;
}

TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
    TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);
    linkNodeLast(p);
    return p;
}

TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    LinkedHashMap.Entry<K,V> q = (LinkedHashMap.Entry<K,V>)p;
    TreeNode<K,V> t = new TreeNode<K,V>(q.hash, q.key, q.value, next);
    transferLinks(q, t);
    return t;
}
// 将节点从链表中删除
void afterNodeRemoval(Node<K,V> e) { // unlink
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}

void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

// 在get方法之后会调用这个，作用是将头结点，放到尾部，第二个节点编程头结点
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```
