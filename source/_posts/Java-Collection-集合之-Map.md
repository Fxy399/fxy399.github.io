---
title: Java Collection 集合之 Map
date: 2021-04-11 16:01:52
tags: Collection
description: HashMap 源码分析以及其他 Map 实现类概述
---
## Map 概述
Map 在集合里是一个接口， 它的实现类常见的有：HashMap、LinkedHashMap、TreeMap、ConcurrentHashMap。
HashMap 的底层数据结构是：数组+链表/红黑树(JDK1.8之后），数组是 HashMap 的主体，链表/红黑树是为了解决哈希冲突；
LinkedHashMap 的底层数据结构是：数组 + 链表 + 双向链表； TreeMap 的底层数据结构是：红黑树；而 ConcurrentHashMap 的底层数据结构也是数组 + 链表/红黑树。

## HashMap 源码分析
### HashMap 的数据结构
为了清晰展示 HashMap 我引用了一张网上的结构图：
![JDK1.8 的 HashMap 数据结构](https://raw.githubusercontent.com/Snailclimb/JavaGuide/master/docs/java/collection/images/jdk1.8%E4%B9%8B%E5%90%8E%E7%9A%84%E5%86%85%E9%83%A8%E7%BB%93%E6%9E%84-HashMap.png)


**先说哈希冲突与链表**
JDK1.8 之前的 HashMap 采用的都是 数组+链表的结构，JDK1.8之后就采用了 数组+链表/红黑树的结构。
上面我已经说了链表是为了解决哈希冲突而使用的，我们将传入的 key 值通过哈希函数算出它的哈希值，再根据哈希值找到 value 在数组中的位置。但是当新增数据的时候哈希函数出现冲突，意思就是数组的某个位置上需要存放超过一个 value，这个时候就需要用到链表来存储多个 key。所以说链表的目的是解决哈希冲突。
但是当哈希冲突严重的时候，数组的某个节点可能会存放一个超长链表，就会提高查找数据的时间复杂度。
这个时候为了提高查询速度，我们就需要用到红黑树了，红黑树的特点是稳定和平衡（最好最坏情况下时间复杂度都为 nlogn）。

**什么时候链表会转化成红黑树**
所以在 JDK1.8 之后的 HashMap 中，当链表长度超过 8 的时候，就会将链表转化为红黑树，提高查询效率，当链表长度低于 8 的时候，红黑树又会退化为链表，可以说是能屈能伸了

### HashMap 是如何初始化的
我们先看源码：
```java
/**
 * The default initial capacity - MUST be a power of two.
 * 默认容量为16，必须为二的幂次方（后面我会讲讲为什么一定要是2的幂）
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
/**
 * The load factor used when none specified in constructor.
 * 看来还是要英语好啊，百度了这段翻译的意思是：如果构造的时候没有指定负载因子大小，就会赋予一个默认的  * 负载因子大小
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```
当我们初始化一个 HashMap 会发生什么呢？ 如果没有指定默认容量以及负载因子大小，首先我们会默认容量初始值为 16，并且在没有设置负载因子的情况下默认负载因子是 0.75，意思是当数据达到容量的百分之 75 的时候，HashMap 就需要扩容，这样做的目的是减少哈希冲突。
**Tips:**如果我们初始化 HashMap 的时候就赋予一个容量，无论传入的数值是多少，最后的容量值都为2的幂次方。
### HashMap 如何 put 数据的
还是先看源码
```java
transient Node<K,V>[] table; //全局的数组链表，HashMap的底层数据结构，新加入的数据都会放入这里
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
//这里有一个 hash 方法，貌似就是用来算 hashcode 的
static final int hash(Object key) {
    int h;
    //获取 key 自带的 hashCode, 并将key 的高16位和低16位进行异或，目的是减少碰撞几率
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
           boolean evict) {
Node<K,V>[] tab; Node<K,V> p; int n, i;
if ((tab = table) == null || (n = tab.length) == 0)
    n = (tab = resize()).length;
    //数组下标的计算方法：(n - 1) & hash，看到这里就懂为什么数组容量必须为2的次幂了
    //如果该数组下标指向的位置为空则直接将新数据插入
if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
else {
    //数组下标指向的位置已经有数据
    Node<K,V> e; K k;
    //如果该位置的 value 与新数据一致，则将旧的 value 值替换为新数据
    if (p.hash == hash &&
        ((k = p.key) == key || (key != null && key.equals(k))))
        e = p;
    //如果该位置数据结构为红黑树则调用 putTreeVal 直接将数据插入红黑树中
    else if (p instanceof TreeNode)
        e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
    else {
    //如果该位置数据结构不为红黑数则将新数据追加在链表后面
        for (int binCount = 0; ; ++binCount) {
            if ((e = p.next) == null) {
                p.next = newNode(hash, key, value, null);
                //当链表大小超过8的时候将链表转成红黑树
                if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                    //调用链表转红黑树方法
                    treeifyBin(tab, hash);
                break;
            }
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                break;
            p = e;
        }
    }
    if (e != null) { // existing mapping for key
        V oldValue = e.value;
        if (!onlyIfAbsent || oldValue == null)
            e.value = value;
        afterNodeAccess(e);
        return oldValue;
    }
}
++modCount;
//键值对数量超过阈值时，则进行扩容
if (++size > threshold)
    resize();
afterNodeInsertion(evict);
return null;
}
//这里是链表转红黑树的方法
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    //如果数组大小超过64则将链表转红黑树
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```
### hash 算法原理以及容量必须为2次幂的原因
要知道 HashMap 的容量为什么需要是2次幂，首先就要了解 HashMap 的底层数据结构以及它的原理。
文章开头说了 HashMap 的底层数据结构为 数组 + 链表/红黑树。我们每次 put 一个数都会为 key 生成一个 hash 值，由于数组大小有限，并且 hash 一般都比较大，无法直接把 hash 作为数组下标，这个时候我们就想到对 hash 做取余运算， 数组下标 index = hash%length,就可以尽量将数据平均分散到数组的各个位置。
**但是**我们看到代码里数组下标的计算方式却是 (n-1)&hash。这是为什么？
我们试着来模拟一下算法，如果 n=16, 那么 n-1 就是 15， 15 的二进制表示是 00000000 00000000 00001111。这样一想就理解为什么数组容量 n 必须为二的次幂了， 因为 n-1 正好是一个 **低位掩码**，如果 (n-1) 与 hash 值做与运算，就正好截取了 hash 的低四位。它的原理跟对 hash 做取余运算是一样的，并且与运算是位运算，使用位运算的效率明显比取余更高，因为底层运算最终都会被转换成位运算。
示例：
```
  00000000 00000000 00001111
& 10101011 10001010 11010101
-----------------------------
  00000000 00000000 00000101      //高位全部归零，只保留低四位
```
那么问题又来了，hash 值分布再松散都好，如果只保留低四位，重复的可能性还是很大的，哈希碰撞的可能性就会贼大。
这个时候，我们就想到 hash 算法，它到底是怎么解决这个问题的。
我们用一张图表示：key.hashCode() ^ (h >>> 16) 的运算过程
![hash算法图解](https://pic1.zhimg.com/80/4acf898694b8fb53498542dc0c5f765a_720w.jpg)
看到这里，我们就明白 hashCode 要与右移16位后的数据做异或运算了，使高 16 位与低 16位做异或运算，就是为了混合原始 hashCode 的高位与低位，以此来加大低位的随机性。而且混合后的低位也保存了部分高位的特征，将高位的信息变相保存了下来。
### 链表和红黑树
从上面的代码可以看到，当链表长度超过8的时候链表会转成红黑树，当红黑树长度低于6的时候又会退化成链表，真可谓是能屈能伸。

## 随便讲讲 LinkedHashMap、TreeMap 与 ConcurrentHashMap
### LinkedkedHashMap 数据结构与应用场景
在前面也提到 LinkedHashMap 底层结构是数组 + 链表 + 双向链表，实际上它继承了 HashMap，在 HashMap 基础上维护了一个双向链表。 我们知道 HashMap 是无序的嘛，所以就搞了个有序的 HashMap 出来，就可以根据插入顺序来遍历了。那如果我既想要 HashMap 的查询速度，又想要能够按照插入顺序来排序就可以用 CoucurrentHashMap 啦。
### TreeMap 的应用场景
TreeMap 的底层数据结构是红黑树，因为树的节点不能为空的嘛，所以 key 不能是 null。我们回想一下红黑树的特点是啥？稳定性嘛，红黑树是平衡树的一种，它的最好最差时间复杂度都是N(logn)，所以比较适合用于对稳定性要求较高的场景中。
### ConcurrentHashMap 概述
线程安全的 Map 有 HashTable、ConcurrentHashMap
但是 HashTable 里面是一堆直接把 synchronized 往上套的方法，性能比较低。
所以我们一般如果对线程安全有要求都是使用 ConcurrentHashMap。 ConcurrentHashMap 通过部分加锁和利用CAS算法来实现同步，在 get 的时候没有加锁，Node 还都使用了 Volatile 修饰。在扩容的时候还会给每个线程分配对应的区间，并且为了防止 putVal 导致数据不一致，会给线程所负责的区间枷锁。
总结语一下：ConcurrentHashMap 的优点就是线程安全，性能比直接套 synchronized 的 HashTable 好多了。

**参考文章：**
[JDK 源码中 HashMap 的 hash 方法原理是什么?](https://www.zhihu.com/question/20733617/answer/111577937)
