## ThreadLocal

`ThreadLocal` 是 Java 中的一个类，用于创建线程局部变量。**每个线程都有自己的变量副本，互不干扰**。`ThreadLocal` 可以用于在多线程环境下保持线程间独立的数据，常见的使用场景包括：

1. **线程安全的数据共享：** `ThreadLocal` 可以用于在多线程环境下安全地共享数据，每个线程拥有自己的数据副本，互不影响。
2. **上下文传递：** 可以使用 `ThreadLocal` 传递上下文信息，避免显式传递参数的麻烦。例如，在Web应用中，可以将用户身份信息、请求信息等存储在 `ThreadLocal` 中，方便在各个层次的代码中访问。
3. **避免传递参数的复杂性：** 在某些情况下，将参数传递给每个方法都会显得繁琐，使用 `ThreadLocal` 可以避免这种复杂性，因为数据被存储在线程本地。
4. **线程池场景：** 在使用线程池时，可以使用 `ThreadLocal` 存储一些线程私有的状态，而不需要担心线程复用时数据混乱。



从 `Thread`类源代码入手

```java
public class Thread implements Runnable {
    //......
    //与此线程有关的ThreadLocal值。由ThreadLocal类维护
    ThreadLocal.ThreadLocalMap threadLocals = null;

    //与此线程有关的InheritableThreadLocal值。由InheritableThreadLocal类维护
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
    //......
}
```

从上面`Thread`类 源代码可以看出`Thread` 类中有一个 `threadLocals` 和 一个 `inheritableThreadLocals` 变量，它们都是 `ThreadLocalMap` 类型的变量,我们可以把 `ThreadLocalMap` 理解为`ThreadLocal` 类实现的定制化的 `HashMap`。默认情况下这两个变量都是 null，只有当前线程调用 `ThreadLocal` 类的 `set`或`get`方法时才创建它们，实际上调用这两个方法的时候，我们调用的是`ThreadLocalMap`类对应的 `get()`、`set()`方法



#### **ThreadLocal的数据结构**

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240305220202288.png" alt="image-20240305220202288" style="zoom: 50%;" />

**每个`Thread`中都具备一个`ThreadLocalMap`，而`ThreadLocalMap`可以存储以`ThreadLocal`为 key ，Object 对象为 value 的键值对。**

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    //......
}
```

比如我们在同一个线程中声明了两个 `ThreadLocal` 对象的话， `Thread`内部都是使用仅有的那个`ThreadLocalMap` 存放数据的，`ThreadLocalMap`的 key 就是 `ThreadLocal`对象，value 就是 `ThreadLocal` 对象调用`set`方法设置的值。

`ThreadLocalMap`有自己的独立实现，可以简单地将它的`key`视作`ThreadLoca<?>`，`value`为代码中放入的值（实际上`key`并不是`ThreadLocal`本身，而是它的一个**弱引用**）。

每个线程在往`ThreadLocal`里放值的时候，都会往自己的`ThreadLocalMap`里存，读也是以`ThreadLocal`作为引用，在自己的`map`里找对应的`key`，从而实现了**线程隔离**。

`ThreadLocalMap`有点类似`HashMap`的结构，只是`HashMap`是由**数组+链表**实现的，而`ThreadLocalMap`中并没有**链表**结构。我们还要注意`Entry`， 它的`key`是`ThreadLocal<?> k` ，继承自`WeakReference`， 也就是我们常说的**弱引用类型。**



#### **ThreadLocal 内存泄露问题是怎么导致的？**

`ThreadLocalMap` 中使用的 key 为 `ThreadLocal` 的弱引用，而 value 是强引用。所以，如果 `ThreadLocal` 没有被外部强引用的情况下，在垃圾回收的时候，**key 会被清理掉，而 value 不会被清理掉。**

这样一来，`ThreadLocalMap` 中就会出现 key 为 null 的 Entry。假如我们不做任何措施的话，value 永远无法被 GC 回收，这个时候就可能会产生内存泄露。`ThreadLocalMap` 实现中已经考虑了这种情况，在调用 `set()`、`get()`、`remove()` 方法的时候，会清理掉 key 为 null 的记录。**使用完 `ThreadLocal`方法后最好手动调用`remove()`方法**

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

**弱引用介绍：**

> 如果一个对象只具有弱引用，那就类似于**可有可无的生活用品**。弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它 所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。
>
> 弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java 虚拟机就会把这个弱引用加入到与之关联的引用队列中。

为了避免 `ThreadLocal` 内存泄露问题，可以采取以下一些建议：

- **及时清理：** 在使用完 `ThreadLocal` 后，及时调用 `remove` 方法清理。可以使用 `try-with-resources` 或者 `finally` 块确保在线程结束时调用 `remove`。





