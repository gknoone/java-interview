# ConcurrentHashMap
<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=6 orderedList=false} -->
<!-- code_chunk_output -->

* [太长不看](#太长不看)
* [实现原理](#实现原理)
* [JDK1.7 实现](#jdk17-实现)
	* [数据结构](#数据结构)
	* [get 方法](#get-方法)
	* [put 方法](#put-方法)
	* [size 方法](#size-方法)
* [JDK1.8 实现](#jdk18-实现)
	* [put 方法](#put-方法-1)
	* [get 方法](#get-方法-1)

<!-- /code_chunk_output -->

## 思维导图

![](assets/ConcurrentHashMap.png)

## 实现原理

由于 `HashMap` 是一个线程不安全的容器，主要体现在容量大于`总量*负载因子`发生扩容时会出现环形链表从而导致死循环。

因此需要支持线程安全的并发容器 `ConcurrentHashMap` 。



## JDK1.7 实现

### 数据结构

![img](assets/ConcurrentHashMap-JDK1-7实现.jpeg)

由 `Segment` 数组、`HashEntry` 数组组成，和 `HashMap` 一样，仍然是数组加链表组成。

`ConcurrentHashMap` 采用了分段锁技术，其中 `Segment` 继承于 `ReentrantLock`。不会像 `HashTable` 那样不管是 `put` 还是 `get` 操作都需要做同步处理，理论上 ConcurrentHashMap 支持 `CurrencyLevel` (Segment 数组数量)的线程并发。每当一个线程占用锁访问一个 `Segment` 时，不会影响到其他的 `Segment`。

### get 方法

`ConcurrentHashMap` 的 `get` 方法是非常高效的，因为整个过程都不需要加锁。

只需要将 `Key` 通过 `Hash` 之后定位到具体的 `Segment` ，再通过一次 `Hash` 定位到具体的元素上。由于 `HashEntry` 中的 `value` 属性是用 `volatile` 关键词修饰的，保证了内存可见性，所以每次获取时都是最新值([volatile 相关知识点](https://github.com/crossoverJie/Java-Interview/blob/master/MD/Threadcore.md#可见性))。

### put 方法

内部 `HashEntry` 类 ：

```
    static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;

        HashEntry(int hash, K key, V value, HashEntry<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }
```

虽然 HashEntry 中的 value 是用 `volatile` 关键词修饰的，但是并不能保证并发的原子性，所以 put 操作时仍然需要加锁处理。

首先也是通过 Key 的 Hash 定位到具体的 Segment，在 put 之前会进行一次扩容校验。这里比 HashMap 要好的一点是：**HashMap 是插入元素之后再看是否需要扩容，有可能扩容之后后续就没有插入就浪费了本次扩容(扩容非常消耗性能)。**

而 ConcurrentHashMap 不一样，它是在将数据插入之前检查是否需要扩容，之后再做插入操作。

### size 方法

每个 `Segment` 都有一个 `volatile` 修饰的全局变量 `count` ,求整个 `ConcurrentHashMap` 的 `size` 时很明显就是将所有的 `count` 累加即可。但是 `volatile` 修饰的变量却不能保证多线程的原子性，所有直接累加很容易出现并发问题。

但如果每次调用 `size` 方法将其余的修改操作加锁效率也很低。所以做法是先尝试两次将 `count` 累加，如果容器的 `count` 发生了变化再加锁来统计 `size`。

至于 `ConcurrentHashMap` 是如何知道在统计时大小发生了变化呢，每个 `Segment` 都有一个 `modCount` 变量，每当进行一次 `put remove` 等操作，`modCount` 将会 +1。只要 `modCount` 发生了变化就认为容器的大小也在发生变化。

## JDK1.8 实现

![img](assets/ConcurrentHashMap-JDK1-8实现.jpeg)

1.8 中的 ConcurrentHashMap 数据结构和实现与 1.7 还是有着明显的差异。

其中抛弃了原有的 Segment 分段锁，而采用了 `CAS + synchronized` 来保证并发安全性。

![img](assets/ConcurrentHashMap1-8code.jpeg)

也将 1.7 中存放数据的 HashEntry 改为 Node，但作用都是相同的。

其中的 `val next` 都用了 volatile 修饰，保证了可见性。

### put 方法

重点来看看 put 函数：

![img](assets/ConcurrentHashMap1-8putcode.jpeg)

- 根据 key 计算出 hashcode 。

- 判断是否需要进行初始化。

- `f` 即为当前 key 定位出的 Node，如果为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功。

- 如果当前位置的 `hashcode == MOVED == -1`,则需要进行扩容。

- 如果都不满足，则利用 synchronized 锁写入数据。

- 如果数量大于 `TREEIFY_THRESHOLD` 则要转换为红黑树。



### get 方法

![img](assets/ConcurrentHashMap1-8getcode.jpeg)

- 根据计算出来的 hashcode 寻址，如果就在桶上那么直接返回值。
- 如果是红黑树那就按照树的方式获取值。
- 都不满足那就按照链表的方式遍历获取值。
