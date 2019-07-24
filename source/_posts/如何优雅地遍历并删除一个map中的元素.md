title: 如何优雅地遍历并删除一个map中的元素
tags:
  - Java
categories:
  - 基础知识
author: ''
date: 2019-03-21 11:18:00
---
最近在实践基于netty造一个http服务器，需要实现`session`功能，需要有一个异步线程定期检查sessionMap中哪些session过期，过期的session需要删除，这个过程需要一边遍历Map一边删除元素，趁此机会探索一下如何优雅地实现这个功能

<!--more-->

## 单线程环境(HashMap)

不考虑多线程的情况，单线程的情况下使用`HashMap`存储并遍历元素有以下方式：

### 对Entry作foreach遍历

对HashMap的Entry作foreach遍历，遍历时如果value为"111"则进行remove：

```java
for (Map.Entry<String, Object> entry : hashMap.entrySet()) {
    if ("111".equals(entry.getValue())) {
        hashMap.remove(entry.getKey());
    }
}
```

显而易见这种情况会触发`ConcurrentModificationException`，因为foreach实际上是调用`EntrySet`的迭代器，也就是`HashMap`内部的实现：`EntryIterator`，在进行遍历时会调用这个迭代器的`next()`方法，最终调用的是`HashIterator`的`nextNode()`方法：

```java
final Node<K,V> nextNode() {
    Node<K,V>[] t;
    Node<K,V> e = next;
    // 如果当前modCount与expectedModCount不匹配则抛异常
    // hashmap的remove()方法会改变modCount，所以这个时候通过此方式遍历当然会报错了
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
	// ...略
    return e;
}
```

`expectedModCount`即预期`modCount`，在初始化迭代器时一同初始化，如果通过迭代器进行遍历时改变了`modCount`就会出现经典的`ConcurrentModificationException`，这个异常在许多其他容器的遍历过程中都会存在，其用意就是让用户另外选择更为安全的遍历方式

由于`hashMap.remove(entry.getKey())`这个操作会更新`modCount`，所以`ConcurrentModificationException`就免不了啦

### 对Entry作foreach遍历（entrySet.forEach(lambda)方式）

使用`EntrySet`自己实现的`forEach()`方法进行遍历，jdk8更新中与lambda表达式一起新加入的，看上去跟上一种方法差不多：

```java
hashMap.entrySet().forEach(entry -> {
    if ("111".equals(entry.getValue())) {
        hashMap.remove(entry.getKey());
    }
});
```

使用该方式遍历并删除元素，效果跟上一种一样，当然如预期一样抛出了`ConcurrentModificationException`异常，`EntrySet`自己的`forEach()`也是检测`modCount`在遍历时有没有改动：

```java
public final void forEach(Consumer<? super Map.Entry<K,V>> action) {
    Node<K,V>[] tab;
    if (action == null)
        throw new NullPointerException();
    if (size > 0 && (tab = table) != null) {
        int mc = modCount;
        // 用for循环进行遍历，并执行lambda过程
        for (int i = 0; i < tab.length; ++i) {
            for (Node<K,V> e = tab[i]; e != null; e = e.next)
                action.accept(e);
        }
        // 遍历完了看看modCount有没有改动，有改动就抛异常
        if (modCount != mc)
            throw new ConcurrentModificationException();
    }
}
```

这种实现方式倒是没有上一种曲折，不过没有达到我们的目的

### 对HashMap作foreach遍历(hashMap.forEach(lambda)方式)

这次直接调用`HashMap`自己的`foreach`方法:

```java
hashMap.forEach((key, value) -> {
    if ("111".equals(value)) {
        hashMap.remove(key);
    }
});
```

更简洁了，充分体验了lambda的好处，不过`ConcurrentModificationException`还是跑不了，继续探索吧

### 显式的调用EntryIterator并EntryIterator.remove()

翻翻`EntryIterator`的源码可以发现他的父类`HashIterator`实现了一个`remove()`方法，想到`List`的各个实现类都实现了自己的迭代器，以支持在遍历过程中对元素进行新增或者删除，举一反三可以想想现在这个`remove()`方法是不是可以达到我们的目的：

```java
HashMap<String, Object> hashMap = new HashMap<>();
hashMap.put("hahah", "111");
hashMap.put("hehehe", "222");
hashMap.put("heiheihei", "333");
System.out.println(hashMap);
Iterator<Map.Entry<String, Object>> entryIterator = hashMap.entrySet().iterator();
// 显示地执行迭代器的迭代方法
while (entryIterator.hasNext()) {
    Map.Entry<String, Object> next = entryIterator.next();
    if ("111".equals(next.getValue())) {
        // 使用HashIterator自己的remove，而不是HashMap的remove
        entryIterator.remove();
    }
}
System.out.println(hashMap);
```

结果如下，未曾抛出任何异常，说明这种方式是切实有效的：

```java
{hahah=111, heiheihei=333, hehehe=222}
{heiheihei=333, hehehe=222}

Process finished with exit code 0
```

`HashIterator`的`remove()`为了支持遍历时删除，对`expectedModCount`做了一些手脚，源码如下：

```java
public final void remove() {
    Node<K,V> p = current;
    if (p == null)
        throw new IllegalStateException();
    // 删除操作还未进行的时候发现modCount被修改了就抛异常
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    current = null;
    K key = p.key;
    removeNode(hash(key), key, null, false, false);
    // 删除操作完成，把expectedModCount修改为最新的modCount
    expectedModCount = modCount;
}
```

在调用完`removeNode`方法（这个过程会修改`modCount`）后，再把`expectedModCount`修改为`removeNode`，保证后续检查`expectedModCount`不会出问题

> 为什么如此设计？
>
> 迭代过程中调用`HashIterator`自己的remove，删除当前迭代指针指向的元素，可以保证这个操作是不受外界影响的（比如其他线程的并发操作），hashMap.remove()就保证不了这个前提

同理，`ListIterator`的设计思路也是基于相同的考虑

### removeIf()

jdk8 引入了一个更为简练的方式：`removeIf`，是`Collection`的default方法，对元素进行遍历，满足条件就进行删除：

```java
hashMap.entrySet().removeIf(next -> "111".equals(next.getValue()));
```

进行该操作完全不需要考虑`ConcurrentModificationException`了，其内部也是调用的对应`Iterator`实现的`remove()`方法，原理跟上一种一致



