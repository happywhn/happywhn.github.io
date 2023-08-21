---
title: ArrayList与LinkedList
categories: [Java,数据结构]
comments: true
---

## ArrayList

**扩容规则**
1. ArrayList() 会使用长度为零的数组，一个空的Object[]
2. ArrayList(int initialCapacity) 会使用指定容量的数组
3. public ArrayList(Collection<? extends E> c) 会使用 c 的大小作为数组容量
4. add(Object o) 首次扩容为 10(DEFAULT_CAPACITY=10)，再次扩容为上次容量的 1.5 倍(oldCapacity + (oldCapacity >> 1))
5. addAll(Collection c) 没有元素时，扩容为 Math.max(10, 实际元素个数)，有元素时为 Math.max(原容量 1.5 倍, 实际元素个数)

*扩容函数*
```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

## LinkedList

## ArrayList 与 LinkedList 对比
**LinkedList**
1. 基于双向链表，无需连续内存
2. 随机访问慢（要沿着链表遍历）
3. 头尾插入删除性能高
4. 占用内存多（指针、以及分配元素数据对齐）

**ArrayList**
1. 基于数组，需要连续内存
2. 随机访问快（指根据下标访问）
3. 尾部插入、删除性能可以，其它部分插入、删除都会移动数据，因此性能会低
4. CPU 的局部性原理，可以利用 CPU 缓存


插入元素：
- 首尾位置：
  - ArrayList 增加元素在不扩展大小的情况下，尾部直接加入，头部需要将全部数据向后移动一个位置，再修改位置0元素
  - LinkedList ：首尾直接定位，添加节点
- 中间位置：
  - ArrayList：可以直接定位，但是需要移动后续元素
  - LinkedList：一个一个遍历定位，添加节点