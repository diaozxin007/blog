---
title: 那些有趣的代码(三)--勤俭持家的 ArrayList
date: 2019-12-05 15:11:02
tags: [java,ArrayList,transient]
---


上周在群里有小盆友问 `transient` 关键字是干什么的。这篇文章就以此为契机介绍一下 `transient` 的作用，以及在 ArrayList 里面的应用。

要了解 transient 我们先聊聊 Java 的序列化。

## 复习序列化

所谓序列化是指，把对象转化为字节流的一种机制。同理，反序列化指的就是把字节流转化为对象。

![serializable](https://img.xilidou.com/img/serializable.jpg)

<!--more-->

- 对于 Java 对象来说，如果使用 JDK 的序列化实现。对象需要实现 `java.io.Serializable` 接口。
- 可以使用 `ObjectOutputStream()` 和 `ObjectInputStream()` 对对象进行序列化和反序列化。
- 序列化的时候会调用 `writeObject()` 方法，把对象转换为字节流。
- 反序列化的时候会调用 `readObject()1 方法，把字节流转换为对象。
- Java 在反序列化的时候会校验字节流中的 `serialVersionUID`  与对象的 `serialVersionUID` 时候一致。如果不一致就会抛出 `InvalidClassException` 异常。官方强烈推荐为序列化的对象指定一个固定的 `serialVersionUID`。否则虚拟机会根据类的相关信息通过一个摘要算法生成，所以当我们改变类的参数的时候虚拟机生成的 `serialVersionUID` 是会变化的。
- `transient` 关键字修饰的变量 **不会** 被序列化为字节流

## 复习ArrayList

1、ArrayList 是基于数组实现的，是一个动态数组，容量支持自动自动增长
2、ArrayList 线程不安全
3、ArrayList 实现了 Serializable，支持序列化

## 勤俭持家

上文我们说到 ArrayList 是基于数组实现，我们看看源码：

```java
    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */

    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * The size of the ArrayList (the number of elements it contains).
     *
     * @serial
     */
    private int size;
```

有几个重要的信息：

- ArraryList 是动态数组，这个 elementData 就是存储对象的数据。
- 这个数组居然使用了 transient 来修饰这个这个存储数据的数组。
- 数组的长度等于 ArrayList 的容量。而不是 ArrayList 的元素数量。
- size 是指的 ArrayList 中元素的数量，不是动态数组的长度。
- size 没有被 transient 修饰，是可以被序列化的。

这，怎么回事。ArrayList 存储数据的数组，居然不需要序列化？

![black](https://img.xilidou.com/img/black.jpg)

莫慌，我们继续往下看代码。上文我们说过，对象的序列化和反序列化是通过调用方法 writeObject() 和 readObject() 完成了，我们发现，ArrayList 自己实现这两个方法看代码：

```java
        /**
         * Save the state of the <tt>ArrayList</tt> instance to a stream (that
         * is, serialize it).
         *
         * @serialData The length of the array backing the <tt>ArrayList</tt>
         *             instance is emitted (int), followed by all of its elements
         *             (each an <tt>Object</tt>) in the proper order.
         */
        private void writeObject(java.io.ObjectOutputStream s)
            throws java.io.IOException{
            // Write out element count, and any hidden stuff
            int expectedModCount = modCount;
            s.defaultWriteObject();

            // Write out size as capacity for behavioural compatibility with clone()
            s.writeInt(size);

            // Write out all elements in the proper order.
            for (int i=0; i<size; i++) {
                s.writeObject(elementData[i]);
            }

            if (modCount != expectedModCount) {
                throw new ConcurrentModificationException();
            }
        }

        /**
         * Reconstitute the <tt>ArrayList</tt> instance from a stream (that is,
         * deserialize it).
         */
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            elementData = EMPTY_ELEMENTDATA;

            // Read in size, and any hidden stuff
            s.defaultReadObject();

            // Read in capacity
            s.readInt(); // ignored

            if (size > 0) {
                // be like clone(), allocate array based upon size not capacity
                int capacity = calculateCapacity(elementData, size);
                SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, capacity);
                ensureCapacityInternal(size);
                Object[] a = elementData;
                // Read in all elements in the proper order.
                for (int i=0; i<size; i++) {
                    a[i] = s.readObject();
                }
            }
        }
```

注意，在 writeObject() 方法中，

```java
      // Write out all elements in the proper order.
      for (int i=0; i<size; i++) {
           s.writeObject(elementData[i]);
      }
```

按需序列化，用了几个下标序列化几个对象。

读取的时候也是:

```java
     for (int i=0; i<size; i++) {
          a[i] = s.readObject();
     }
```

有几个读几个。

总结一下：

- 被 `transient` 修饰的变量不会被序列化。
- ArrayList 的底层数组 `elementData` 被 `transient` 修饰，不会直接被序列化。
- 为了实现 ArrayList 元素的序列化，ArrayList 重写了 `writeObject()` 和 `readObject()` 方法。
- 按需序列化数组，只序列化存在的数据，而不是序列化整个 `elementData` 数组。

用多少，序列化多少，真是勤俭持家的 ArrayList。

## 有趣的代码系列

[那些有趣的代码(一)--有点萌的 Tomcat 的线程池](https://xilidou.com/2019/10/15/tomcat-threadpool/)
[那些有趣的代码(二)--偏不听父母话的 Tomcat 类加载器](https://xilidou.com/2019/10/27/tomcat-classloader/)
