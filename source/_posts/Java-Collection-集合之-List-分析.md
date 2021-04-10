---
title: Java Collection 集合之 List 分析
date: 2021-04-10 12:48:44
tags: Collection
description: List 各实现类的源码分析及使用场景分析
---

## 前情
> List 在 Java 里面是一个接口， 常见实现类有 ArrayList， LinkedList。
初级 Java 开发面试中经常会遇到 ArrayList 相关的问题。例如 ArrayList 如何实现动态扩容，LinkedList 和 ArrayList 的区别，线程安全的 List 有什么。

## ArrayList 源码解析

### 初始化一个 ArrayList 数组

> 当我们用 new ArrayList() 初始化一个数组, 构建函数会默认创建一个空 Object[] 数组

```
transient Object[] elementData;
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

### 往 ArrayList 中添加元素

当我们第一次往数组中添加元素时，会将数组大小默认初始化为10, 之后每一次添加新元素时都会判断数组是否已满，如果数组已满则会使用 grow 方法将创建大小为原数组1.5倍大的新数组，然后将旧数组的数据通过 Arrays.copyof 方法复制到新数组, 最后将才将新数据插入。

下面我们来看看数组是怎么做到动态扩容的：
```
//modCount 参数记录修改次数
//当使用迭代器遍历数组的时候,用来检测列表中的元素是否发生变化(remove 或 add),主要是在多线程环境下使用,防止当一个线程迭代遍历的时候,另一个线程修改了这个列表,因为ArrayList是非线程安全的
protected transient int modCount = 0;

public boolean add(E e) {
    ensureCapacityInternal(size + 1);
    elementData[size++] = e;
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // 判断数组大小是否已满并扩容
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

private void grow(int minCapacity) {
    // 新数组大小扩容为原来的1.5倍
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // 将原数组复制到新数组
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```
**提示：**由于数组是动态扩容的，每次扩容都要复制一次数据，因此在开发中如果能事先就知道数组大小，我们最好在初始化数组的时候就给一个固定的大小。
### modCount 作用
根据上面对 ArrayList 的分析可以得知它并不是线程安全的，ArrayList 有一个全局变量 modCount，用以记录数组修改次数，每次迭代都会判断数组是否已经被修改，如果 A 线程对数组进行迭代的过程中， B 线程修改了数组，那么就会抛出 ConcurrentModificationException 异常。
```
//ArrayList 内部实现了 Iterator 迭代器
private class Itr implements Iterator<E> {
    int cursor;       // index of next element to return
    int lastRet = -1; // index of last element returned; -1 if no such
    int expectedModCount = modCount; //初始化 Iterator 时将当前 modCount 赋值给 expectedModCount

    Itr() {}

    public boolean hasNext() {
        return cursor != size;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        //每次迭代都会判断数组是否已经被修改
        checkForComodification();
        int i = cursor;
        if (i >= size)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i + 1;
        return (E) elementData[lastRet = i];
    }
    //此方法判断数组是否已经被修改
    final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    }
}
```
## 不推荐使用 Vector 的原因
Vector 底层数据结构是数组，并且是线程安全的，但是很少使用。
原因是：
    1. Vector 的方法都使用 synchronized 关键字修饰，synchronized 关键字在 JDK1.6以前是重量级锁，十分笨重，即使现在 synchronized 被优化，但是加锁必然就会牺牲性能。
而且最骚的是 Vector 全套方法都加了锁，连 get() 方法都不放过，可以说是性能极低。
    2. 而且 Vector 每次扩容都是将原数组大小扩大2倍，而 ArrayList 仅仅是1.5倍。
```
//就问你骚不骚
public synchronized E get(int index) {
    if (index >= elementCount)
        throw new ArrayIndexOutOfBoundsException(index);

    return elementData(index);
}
```

## 线程安全的 List 推荐使用 CopyOnWriteArrayList
CopyONWriteArrayList 是线程安全的 List， 底层是通过复制数组实现的。
### CopyOnWrite 思想
CopyOnWrite 顾名思义就是**写入时复制**。
意思是当有人要修改这个资源时，会复制一个副本出来进行修改，修改完之后再把这个副本指向原资源。
文件系统中也有用到 cow 思想，例如当修改 A 的内容，会将 A 复制一份到 B， 当B的操作完成时，再将 A 的指针指向 B，A 原来的数据就会被回收掉。 这样做的好处是能保证数据最终的一致性，比如当突然断电时，原来 A 的内容还在。
### CopyOnWriteArrayList add 方法源码分析
我们先放上一段源码：
```
final transient ReentrantLock lock = new ReentrantLock();
public boolean add(E e) {
    final ReentrantLock lock = this.lock;                          //获取全局锁
    lock.lock();
    try {
        Object[] elements = getArray();                         
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);  //复制一份数据到新数组
        newElements[len] = e;                                     //将新数据加到新数组末尾
        setArray(newElements);                                    //将指针指向新数组
        return true;
    } finally {
        lock.unlock();                                            //释放锁
    }
}

final void setArray(Object[] a) {
    array = a;
}
```
我们可以看到 add 方法完美实践了 cow 思想，首先获取全局锁锁住，然后复制一份数据到新数组，并将 add 的新数据加到新数组末尾，添加完数据之后再将原数组指针指向新数组，最后再释放锁。这样就完成了一次数据的添加。
### CopyOnWriteArrayList 的缺点
1. 每次添加数据都需要上锁，并且还要复制一份数据，影响性能。
2. 只能保证数据最终一致性，但是没法抱枕数据实时性。比如当 A 线程正在 add 数据，数组指针仍然指向旧数组的时候，我们获取到的数据其实还是 add 前的数据。

## ArrayList 和 LinkedList 使用场景分析
面试经常会遇到诸如：ArrayList 跟 LinkedList 的区别，已经各自的应用场景等问题。
1.  ArrayList 的底层是 Object[] 数组， 而 LinkedList 的底层是双向链表。
2. 数组能够根据索引来查找数据， 数组在内存中是连续存储的，如果对数组进行增删就需要将增删节点后的数据全部后移或者前移。
3. 链表查找数据时需要从头到尾遍历一遍，不过链表再内存中的存储是不连续的，链表每个节点d保存着头尾节点的指针，如果对数据进行增删只需要修改头尾节点指针就OK
**总结:**
如果内存不足，增删操作较多时最好使用链表。
如果内存充足，需要根据索引查找数据，增删数据较少，或者只在尾部增删数据时最好使用数组。
像我们在日常开发中，可能使用得比较多的都是 ArrayList ，因为遍历需求比增删需求多，而且就算需要增删大部分情况下都是在尾部增删就完事了。而且现代 CPU 对内存可以块操作，Arrays.copyof 方法被优化过，因此增删的性能影响并不会很大。
