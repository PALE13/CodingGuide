## **AQS**

AQS队列同步器,用来构建锁的基础框架,Lock实现类都是基于AQS实现的

AQS是基于模板方法模式进行设计的,所以锁的实现需要继承AQS并重写它指定的方法

AQS内部定义了一个FIFO的队列来实现线程的同步,同时还定义了同步状态来记录锁的信息

AQS的模板方法,将管理同步状态的逻辑提炼出来形成标准流程,这些方法主要包括：独占式获取同步状态、独占式释放同步状态、共享式获取同步状态、共享式释放同步状态

AQS 定义两种资源共享方式：`Exclusive`（独占，只有一个线程能执行，如`ReentrantLock`）和`Share`（共享，多个线程可同时执行，如`Semaphore`/`CountDownLatch`）

一般来说，自定义同步器的共享方式要么是独占，要么是共享，他们也只需实现`tryAcquire-tryRelease`、`tryAcquireShared-tryReleaseShared`中的一种即可。但 AQS 也支持自定义同步器同时实现独占和共享两种方式，如`ReentrantReadWriteLock`



### **AQS 的原理**

AQS 核心思想是，如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制 AQS 是用 **CLH 队列锁** 实现的，即将暂时获取不到锁的线程加入到队列中。

CLH(Craig,Landin,and Hagersten) 队列是一个虚拟的双向队列（虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系）。AQS 是将每条请求共享资源的线程封装成一个 CLH 锁队列的一个结点（Node）来实现锁的分配。在 CLH 同步队列中，**一个节点表示一个线程，它保存着线程的引用（thread）**、 当前节点在队列中的状态（waitStatus）、前驱节点（prev）、后继节点（next）

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240324164942882.png" alt="image-20240324164942882" style="zoom: 50%;" />

AQS 使用 **int 成员变量 `state` 表示同步状态**，通过内置的 **FIFO 线程等待/等待队列** 来完成获取资源线程的排队工作。

`state` 变量由 `volatile` 修饰，用于展示当前临界资源的获锁情况。

```java
// 共享变量，使用volatile修饰保证线程可见性
private volatile int state;
```

另外，状态信息 `state` 可以通过 `protected` 类型的`getState()`、`setState()`和`compareAndSetState()` 进行操作。并且，这几个方法都是 `final` 修饰的，在子类中无法被重写。

```java
//返回同步状态的当前值
protected final int getState() {
     return state;
}
 // 设置同步状态的值
protected final void setState(int newState) {
     state = newState;
}
//原子地（CAS操作）将同步状态值设置为给定值update如果当前同步状态的值等于expect（期望值）
protected final boolean compareAndSetState(int expect, int update) {
      return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}

```

以可重入的互斥锁 `ReentrantLock` 为例，它的内部维护了一个 `state` 变量，用来表示锁的占用状态。**`state` 的初始值为 0，表示锁处于未锁定状态。当线程 A 调用 `lock()` 方法时，会尝试通过 `tryAcquire()` 方法独占该锁，并让 `state` 的值加 1。**如果成功了，那么线程 A 就获取到了锁。如果失败了，那么线程 A 就会被加入到一个等待队列（CLH 队列）中，直到其他线程释放该锁。假设线程 A 获取锁成功了，释放锁之前，A 线程自己是可以重复获取此锁的（`state` 会累加）。这就是可重入性的体现：一个线程可以多次获取同一个锁而不会被阻塞。但是，这也意味着，一个线程必须释放与获取的次数相同的锁，才能让 `state` 的值回到 0，也就是让锁恢复到未锁定状态。只有这样，其他等待的线程才能有机会获取该锁。



### **CHL锁**

#### **自旋锁缺点**

自旋锁实现简单，同时避免了操作系统进程调度和线程上下文切换的开销，但他有两个缺点：

- 第一个是锁饥饿问题。在锁竞争激烈的情况下，可能存在一个线程一直被其他线程”插队“而一直获取不到锁的情况。
- 第二是性能问题。在实际的多处理上运行的自旋锁在锁竞争激烈时性能较差。

CLH 锁是对自旋锁的一种改进，有效的解决了以上的两个缺点。首先它将线程组织成一个队列，保证先请求的线程先获得锁，避免了饥饿问题。其次锁状态去中心化，让每个线程在不同的状态变量中自旋，这样当一个线程释放它的锁时，只能使其后续线程的高速缓存失效，缩小了影响范围，从而减少了 CPU 的开销。

CLH 锁数据结构很简单，类似一个链表队列，**所有请求获取锁的线程会排列在链表队列中，自旋访问队列中前一个节点的状态。当一个节点释放锁时，只有它的后一个节点才可以得到锁。**CLH 锁本身有一个队尾指针 Tail，它是一个原子变量，指向队列最末端的 CLH 节点。每一个 CLH 节点有两个属性：所代表的线程和标识是否持有锁的状态变量。当一个线程要获取锁时，它会对 Tail 进行一个 getAndSet 的原子操作。该操作会返回 Tail 当前指向的节点，也就是当前队尾节点，然后使 Tail 指向这个线程对应的 CLH 节点，成为新的队尾节点。入队成功后，该线程会轮询上一个队尾节点的状态变量，当上一个节点释放锁后，它将得到这个锁。



#### **CLH 优缺点分析**

CLH 锁作为自旋锁的改进，有以下几个优点：

1. 性能优异，获取和释放锁开销小。CLH 的锁状态不再是单一的原子变量，而是分散在每个节点的状态中，降低了自旋锁在竞争激烈时频繁同步的开销。在释放锁的开销也因为不需要使用 CAS 指令而降低了。
2. 公平锁。先入队的线程会先得到锁。
3. 实现简单，易于理解。
4. 扩展性强。下面会提到 AQS 如何扩展 CLH 锁实现了JUC 包下各类丰富的同步器。

当然，它也有两个缺点：第一是因为有自旋操作，当锁持有时间长时会带来较大的 CPU 开销。第二是基本的 CLH 锁功能单一，不改造不能支持复杂的功能。

在原始版本的 CLH 锁中，节点间甚至都没有互相链接。但是，通过在节点中显式地维护前驱节点，CLH 锁就可以处理“超时”和各种形式的“取消”：如果一个节点的前驱节点取消了，这个节点就可以滑动去使用前面一个节点的状态字段。对于通过自旋获取锁的 CLH 锁来说，只需要显式的维护前驱节点就可以实现取消功能，如下图所示：

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240324222041328.png" alt="image-20240324222041328" style="zoom:50%;" />

### **AQS 对 CLH 队列锁的改造**

针对 CLH 的缺点，AQS 对 CLH 队列锁进行了一定的改造。针对第一个缺点，**AQS 将自旋操作改为阻塞线程操作**。针对第二个缺点，AQS 对 CLH 锁进行改造和扩展，原作者 Doug Lea 称之为“CLH 锁的变体”。下面将详细讲 AQS 底层细节以及对 CLH 锁的改进。AQS 中的对 CLH 锁数据结构的改进主要包括三方面：**扩展每个节点的状态、显式的维护前驱节点和后继节点以及诸如出队节点显式设为 null** 等辅助 GC 的优化。正是这些改进使 AQS 可以支撑 j.u.c 丰富多彩的同步器实现。

**显示的添加了前后指针**

因为 AQS 用阻塞等待替换了自旋操作，线程会阻塞等待锁的释放，不能主动感知到前驱节点状态变化的信息。AQS 中显式的维护前驱节点和后继节点，**需要释放锁的节点会显式通知下一个节点解除阻塞**，如下图所示，T1 释放锁后主动唤醒 T2，使 T2 检测到锁已释放，获取锁成功

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240324222229937.png" alt="image-20240324222229937" style="zoom:50%;" />



 **扩展每个节点的状态**

AQS 每个节点的状态如下所示，在源码中如下所示：

```
volatile int waitStatus;
```

AQS 同样提供了该状态变量的原子读写操作，但和同步器状态不同的是，节点状态在 AQS 中被清晰的定义，如下表所示

| 状态名    | 枚举值 | 描述                                     |
| --------- | ------ | ---------------------------------------- |
| SIGNAL    | -1     | 表示队列下一个线程在等待其唤醒           |
| PROPAGATE | -3     | 应将 releaseShared 传播到其他节点        |
| CONDITION | -2     | 该节点位于条件队列，不能用于同步队列节点 |
| CANCELLED | 1      | 由于超时、中断或其他原因，该节点被取消   |
| 0         | 0      | 结点初始默认值或已经被释放               |



### **AQS独占模式分析**

#### **请求锁**

```java
public final void acquire(int arg) {
        //先去尝试获取锁,如果没有获取成功，就在CLH队列中增加一个当前线程的节点，表示等待抢占
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

**tryAcquire方法**

表示线程尝试获得锁，该方法只有一行实现 throw new UnsupportedOperationException()，意图很明显，AQS规定继承类必须重写tryAcquire方法，否则就直接抛出UnsupportedoperationException。那么为什么这里一定需要上层自己实现？因为尝试获取锁这个操作中可能包含某些业务自定义的逻辑，比如是否“可重入”等。

**如果tryAcquire返回true，说明已经获得锁，直接返回；如果返回false，则执行acquireQueued(addWaiter(Node.EXCLUSIVE), arg)，将线程加入等待队列**



#### **将线程加入队列**

假如tryAcquire返回false，说明需要排队，那么就会调用acquireQueued(addWaiter(Node.EXCLUSIVE),arg)，acquireQueued 方法其中嵌套了addWaiter

**addWaiter方法**

先来看addWaiter，**当线程第一次获取锁失败时，会调用addWaiter方法加入到队列中**，并将tail指向该结点，这是一个原子性的操作

```java
 //AbstractQueuedSynchronizer.addWaiter()
  private Node addWaiter(Node mode) {
    // 初始化一个节点，用于保存当前线程
        Node node = new Node(Thread.currentThread(), mode);
        // 当CLH队列不为空的视乎，直接在队列尾部插入一个节点
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            //如果pred还是尾部(即没有被其他线程更新)，则将尾部更新为node节点(即当前线程快速设置成了队尾)
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        // 当CLH队列为空的时候，调用enq方法初始化队列
        enq(node);
        return node;
    }
```

**enq方法**

将该node加入到尾节点

```java
    private Node enq(final Node node) {
        //在一个循环里不停的尝试将node节点插入到队尾里
        for (;;) {
            Node t = tail;
            if (t == null) { // 初始化节点，头尾都指向一个空节点
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

假设当前线程0占有锁，线程1占有锁失败，此时会创建一个当前线程的结点Node，然后执行enq

**如果队列未初始化(tail== null)，那么就尝试初始化，新建一个结点，并将头结点和尾结点指向该结点（该结点不代表任何线程，只是作为头节点去唤醒后继线程结点）**

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240509181546362.png" alt="image-20240509181546362" style="zoom:50%;" />

队列初始化后的状态如上图

此时会继续执行for循环，**然后使用CAS将新的线程的结点插入到队列尾部**

因为使用的是CAS对为节为如果尾插节点失败，那么就不断重试，直到插入成功为止

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240509182055928.png" alt="image-20240509182055928" style="zoom:50%;" />

最后返回插入成功的结点





#### **判断能否获取锁**

**acquireQueued方法**

```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            //在一个循环里不断等待前驱节点执行完毕
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {// 通过tryAcquire获得锁，如果获取到锁，说明头节点已经释放了锁
                    setHead(node);//将当前节点设置成头节点
                    p.next = null; // help GC//将上一个节点的next变量被设置为null，在下次GC的时候会清理掉
                    failed = false; //将failed标记设置成false, 表示拿到了锁
                    return interrupted;
                }
                //中断
                if (shouldParkAfterFailedAcquire(p, node) && // 是否需要阻塞
                    parkAndCheckInterrupt())// 阻塞，返回线程是否被中断
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

acquireQueued方法内定义了一个局部变量failed，初始值为true，意思是默认失败。还有个变量interrupted，初始值为false，意思是等待锁的过程中当前线程没有被中断。

当前置节点为head，**说明当前节点有权限去尝试拿锁**，这是一种约定。**如果tryAcquire返回true，代表拿到了锁**，那么顺理成章，函数返回。**如果tryAcquire为false，就要判断是否要将挂起该线程**

当前置节点不是head，即当前节点没有拿锁的权限或拿锁失败，那么将会进入shouldParkAfterFailedAcquire判断是否需要挂起(park)，方法的参数是pred Node和当前Node的引用。**若需要挂起，interrupted将会设置为ture，终止死循环**

当进入阻塞阶段，会进入 **parkAndCheckInterrupt()** 方法，则会调用 **LockSupport.park(this)** 将当前线程挂起





#### **判断是否需要挂起**

```java
/**
 * 确保当前结点的前驱结点的状态为SIGNAL
 * SIGNAL意味着线程释放锁后会唤醒后面阻塞的线程
 * 只有确保能够被唤醒，当前线程才能放心的阻塞。
 */
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
           //如果前驱节点状态为SIGNAL
           //表明当前线程需要阻塞，因为前置节点承诺执行完之后会通知唤醒当前节点
            return true;
        if (ws > 0) {//ws > 0 代表前驱节点取消了
            do {
                node.prev = pred = pred.prev;//不断的把前驱取消了的节点移除队列
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
         //初始化状态，将前驱节点的状态设置成SIGNAL
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

**如果前置结点为0，将前置结点状态置为SIGNAL，表示该线程需要由前置结点唤醒**

当线程1刚加入进来且获取锁失败时，会执行shouldParkAfterFailedAcquire(p, node)方法，因为线程1的前置结点头节点为初始化状态0，**故需要用CAS将其修改为SIGNAL状态**，然后返回false，重试外层逻辑

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240509162824092.png" alt="image-20240509162824092" style="zoom:50%;" />

**若pred的waitSatus为SIGNAL**，说明前置结点可以唤醒该线程，所以当前线程可以挂起休息，返回true，之后**会调用parkAndCheckInterrupt将该线程挂起**

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240509163115625.png" alt="image-20240509163115625" style="zoom:50%;" />

如果ws大于0，说明pred的waitSatus是CANCEL，所以可以将其从队列中删除。**这里通过从后向前搜索**，将pred指向搜索过程中第一个waitSatus为非CANCEL的节点。相当于链式地删除被CANCEL的节点。然后返回false，代表当前节点不需要挂起，**因为pred指向了新的Node，需要重试外层的逻辑。**



#### **释放锁**

```java
public final boolean release(int arg) {
  		//尝试释放锁
        if (tryRelease(arg)) {
            Node h = head;  //
            if (h != null && h.waitStatus != 0)//waitStatus!=0表明处于CANCEL状态或者是SIGNAL。也就是说waitStatus不为0表示它的后继在等待唤醒
                unparkSuccessor(h);
               //成功返回true
            return true;
        }
        //否则返回false
        return false;
}
```

如果成功释放了锁，就需要判断头节点的状态是否需要唤醒后续线程



#### **判断是否唤醒后续线程**

主要是通过unparkSuccessor来唤醒后续线程

```java
private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
         //如果waitStatus < 0, 即为SIGNAL, 将该结点状态置为0
        if (ws < 0)  compareAndSetWaitStatus(node, ws, 0);
       //若后续节点为空或已被cancel，则从尾部开始找到队列中第一个waitStatus<=0，即未被cancel的节点
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

**不需要唤醒后续线程的情况有**

头节点为空，表示不需要唤醒后续线程

头节点状态为0，表示新加入的线程刚加入AQS队列并尝试获取锁

头节点的状态为0，且新加入的线程已经获取到锁

![image-20240509180808110](https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240509180808110.png)



**需要唤醒的情况**

获取head的waitStatus，如果<0，即为SIGNAL，那么将其置为0，表示锁已释放，然后唤醒后续等待线程

如果后续节点为null或者处于CANCELED状态，那么从后往前搜索，**找到除了head外最靠前且非CANCELED状态的Node，对其进行唤醒，让它起来尝试拿锁**，唤醒的线程会回到acquireQueued(final Node node, int arg)方法的逻辑重新执行**tryAcquire(arg)**

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240509163115625.png" alt="image-20240509163115625" style="zoom:50%;" />







## **自定义同步器**

**同步器的设计是基于模板方法模式的**：如果需要自定义同步器一般的方式是这样（模板方法模式很经典的一个应用）：

1. 使用者继承 `AbstractQueuedSynchronizer` 并重写指定的方法。
2. 将 AQS 组合在自定义同步组件的实现中，并调用其模板方法，而这些模板方法会调用使用者重写的方法。

这和我们以往通过实现接口的方式有很大区别，这是模板方法模式很经典的一个运用。

AQS 使用了模板方法模式，自定义同步器时需要重写下面几个 AQS 提供的钩子方法：

```java
//独占方式。尝试获取资源，成功则返回true，失败则返回false。
protected boolean tryAcquire(int)
//独占方式。尝试释放资源，成功则返回true，失败则返回false。
protected boolean tryRelease(int)
//共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
protected int tryAcquireShared(int)
//共享方式。尝试释放资源，成功则返回true，失败则返回false。
protected boolean tryReleaseShared(int)
//该线程是否正在独占资源。只有用到condition才需要去实现它。
protected boolean isHeldExclusively()
```

**什么是钩子方法呢？** 钩子方法是一种被声明在抽象类中的方法，一般使用 `protected` 关键字修饰，它可以是空方法（由子类实现），也可以是默认实现的方法。模板设计模式通过钩子方法控制固定步骤的实现。





### **ReentrantLock**

`ReentrantLock` 实现了 `Lock` 接口，是一个可重入且独占式的锁，和 `synchronized` 关键字类似。不过，`ReentrantLock` 更灵活、更强大，增加了轮询、超时、中断、公平锁和非公平锁等高级功能。

```java
private volatile int state
```

**它的内部维护了一个 `state` 变量**，用volatile修饰，保证了其可见性，用来表示锁的占用状态。**`state` 的初始值为 0**，表示锁处于未锁定状态。当线程 A 调用 `lock()` 方法时，会尝试通过 `tryAcquire()` 方法独占该锁，**并让 `state` 的值加 1**。如果成功了，那么线程 A 就获取到了锁。如果失败了，那么线程 A 就会被加入到一个等待队列（CLH 队列）中，直到其他线程释放该锁。

**state涉及到三个方法**

![image-20240509154744601](https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240509154744601.png)



```java
public class ReentrantLock implements Lock, java.io.Serializable {}
```

`ReentrantLock` 里面有一个内部类 `Sync`，**`Sync` 继承 AQS（`AbstractQueuedSynchronizer`），添加锁和释放锁的大部分操作实际上都是在 `Sync` 中实现的。`Sync` 有公平锁 `FairSync` 和非公平锁 `NonfairSync` 两个子类**

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240508232856738.png" alt="image-20240508232856738" style="zoom: 67%;" />

`ReentrantLock` 默认使用非公平锁，也可以通过构造器来显式的指定使用公平锁。

```java
// 传入一个 boolean 值，true 时为公平锁，false 时为非公平锁
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

#### **NonfairSync**

**lock()方法**

当我们调用 **ReentrantLock** 的 **lock()** 方法的时候，实际上是调用了 **NonfairSync** 的 **lock()** 方法

```java
static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
         //这个方法先用CAS操作，去尝试抢占该锁
         // 快速尝试将state从0设置成1，如果state=0代表当前没有任何一个线程获得了锁
            if (compareAndSetState(0, 1))
             //state设置成1代表获得锁成功
             //如果成功，就把当前线程设置在这个锁上，表示抢占成功,在重入锁的时候需要
                setExclusiveOwnerThread(Thread.currentThread());
            else
             //如果失败，则调用 AbstractQueuedSynchronizer.acquire() 模板方法，等待抢占。
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```

**非公平性：**当程序调用lock的时候，**会先进行一次CAS的尝试，当尝试获取锁失败时，调用acquire**。在acquire内部，首先会调用一次tryAcquire，而nonfairTryAcquire会直接尝试获取锁，如果锁被占用且不可重入，那么就会继续执行AQS中后续的排队流程。虽然只有那么两次尝试抢占，但也体现出了非公平性。

调用 **acquire(1)** 实际上使用的是 **AbstractQueuedSynchronizer** 的 **acquire()** 方法

**acquire()** 会先调用 **tryAcquire()** 这个钩子方法去尝试获取锁，这个方法就是在 NonfairSync.tryAcquire()下的 nonfairTryAcquire()

```java
//一个尝试插队的过程
  final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            //获取state值
            int c = getState();
            //比较锁的状态是否为 0，如果是0，当前没有任何一个线程获取锁
            if (c == 0) {
             //则尝试去原子抢占这个锁（设置状态为1，然后把当前线程设置成独占线程）
                if (compareAndSetState(0, acquires)) {
                 // 设置成功标识独占锁
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //如果当前锁的状态不是0 state!=0,就去比较当前线程和占用锁的线程是不是一个线程
            else if (current == getExclusiveOwnerThread()) {
             //如果是，增加状态变量的值，从这里看出可重入锁之所以可重入，就是同一个线程可以反复使用它占用的锁
                int nextc = c + acquires;
                //重入次数太多，大过Integer.MAX
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            //如果以上两种情况都不通过，则返回失败false
            return false;
        }
```

**可重入性：**一个线程可以不用释放而重复获取一个锁n次，只是在释放的时候也需要相应释放n次。当锁已经被占用，如果占用锁的线程不是当前线程，那么代表获取锁失败，返回false。如果正是自己，满足可重入性的条件，使用nextc这个值来记录重入的次数，因为释放锁的时候也要释放相应次数。

**tryAcquire()** 一旦返回 **false**，就会则进入 **acquireQueued()** 流程，也就是基于CLH队列的抢占模式，在CLH锁队列尾部增加一个等待节点，这个节点保存了当前线程，通过调用 **addWaiter()** 实现，这里需要考虑初始化的情况，在第一个等待节点进入的时候，需要初始化一个头节点然后把当前节点加入到尾部，后续则直接在尾部加入节点



**unlock()方法**

```java
public void unlock() {
        sync.release(1);
}

public final boolean release(int arg) {
  //尝试在当前锁的锁定计数（state）值上减1，
        if (tryRelease(arg)) { //这里tryRelease成功后，自旋线程会马上获取锁，并设置为头
            Node h = head;
            if (h != null && h.waitStatus != 0) //waitStatus !=0 表明处于CANCEL状态或者是SIGNAL，表示下一个线程在等待其唤醒。也就是说waitStatus不为零表示它的后继在等待唤醒。
                unparkSuccessor(h);
               //成功返回true
            return true;
        }
        //否则返回false
        return false;
}


protected final boolean tryRelease(int releases) {
     int c = getState() - releases;
    //判断是否为当前线程在调用，不是抛出IllegalMonitorStateException异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
    boolean free = false;
    //c == 0,释放该锁，同时将当前所持有线程设置为null
    if (c == 0) {
          free = true;
          setExclusiveOwnerThread(null);
    }
    //设置state
    setState(c);
    return free;
 }


private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            // 从后往前找到离head最近，而且waitStatus <= 0 的节点
            // 其实在ReentrantLock中，waitStatus应该只能为0和-1，需要唤醒的都是-1(Node.SIGNAL)
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null) 
            LockSupport.unpark(s.thread);// 唤醒挂起线程
}

```

调用 **unlock()** 方法，其实是直接调用 **AbstractQueuedSynchronizer.release()** 操作。

进入 **release()** 方法，内部先尝试 **tryRelease()** 操作，主要是去除锁的独占线程，然后将状态减一，这里减一主要是考虑到可重入锁可能自身会多次占用锁，只有当状态变成0，才表示完全释放了锁。

如果 **tryRelease** 成功，则将CHL队列的头节点的状态设置为0，然后唤醒下一个非取消的节点线程。

一旦下一个节点的线程被唤醒，被唤醒的线程就会进入**acquireQueued()** 代码流程中，去抢占获取锁

注意：唤醒的线程可以继续执行**acquireQueued()** 去竞争锁，但并不意味着一定能获取锁，依然要和其他





#### **FairSync**

FairSync相对来说就简单很多，只有重写的两个方法跟NonfairSync不同

使用lock时并不是直接CAS来判断是否能获取锁，而是直接调用acquire

```java
final void lock() {
    acquire(1);
}

protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&// 没有前驱节点了
            compareAndSetState(0, acquires)) {// 而且没有锁
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

在tryAcquire方法中，首先判断锁是否空闲，**如果空闲，此时并不是直接尝试通过CAS获取锁，而是需要判断是否存在前置等待节点。**如果不存在，那说明，在队列中确实已经轮到当前线程尝试获取锁，然后把该结点设置为队头

否则tryAcquire返回false，当前线程将会执行AQS中的后续等待逻辑。这里就体现出了公平性。



### **Semaphore**

`synchronized` 和 `ReentrantLock` 都是一次只允许一个线程访问某个资源，而`Semaphore`(信号量)可以用来控制同时访问特定资源的线程数量。

Semaphore 的使用简单，我们这里假设有 N(N>5) 个线程来获取 `Semaphore` 中的共享资源，下面的代码表示同一时刻 N 个线程中只有 5 个线程能获取到共享资源，其他线程都会阻塞，只有获取到共享资源的线程才能执行。等到有线程释放了共享资源，其他阻塞的线程才能获取到。

```java
// 初始共享资源数量
final Semaphore semaphore = new Semaphore(5);
// 获取1个许可
semaphore.acquire();
// 释放1个许可
semaphore.release();
```

当初始的资源个数为 1 的时候，`Semaphore` 退化为排他锁。

`Semaphore` 有两种模式：。

- **公平模式：** 调用 `acquire()` 方法的顺序就是获取许可证的顺序，遵循 FIFO；
- **非公平模式：** 抢占式的。

`Semaphore` 对应的两个构造方法如下：

```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}

public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```

**这两个构造方法，都必须提供许可的数量，第二个构造方法可以指定是公平模式还是非公平模式，默认非公平模式。**

`Semaphore` 通常用于那些资源有明确访问数量限制的场景比如限流（仅限于单机模式，实际项目中推荐使用 Redis +Lua 来做限流）。



#### **原理**

`Semaphore` 是共享锁的一种实现，**它默认构造 AQS 的 `state` 值为 `permits`**，你可以将 `permits` 的值理解为许可证的数量，只有拿到许可证的线程才能执行。

调用`semaphore.acquire()` ，线程尝试获取许可证，如果 `state >= 0` 的话，则表示可以获取成功。**如果获取成功的话，使用 CAS 操作去修改 `state` 的值 `state=state-1`**。如果 `state<0` 的话，则表示许可证数量不足。此时会创建一个 Node 节点加入阻塞队列，挂起当前线程。

```java
/**
 *  获取1个许可证
 */
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
/**
 * 共享模式下获取许可证，获取成功则返回，失败则加入阻塞队列，挂起线程
 */
public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
      throw new InterruptedException();
        // 尝试获取许可证，arg为获取许可证个数，当可用许可证数减当前获取的许可证数结果小于0,则创建一个节点加入阻塞队列，挂起当前线程。
    if (tryAcquireShared(arg) < 0)
      doAcquireSharedInterruptibly(arg);
}
```

以非公平模式（`NonfairSync`）的为例，看看 `tryAcquireShared` 方法的实现。

```java
// 共享模式下尝试获取资源(在Semaphore中的资源即许可证):
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}

// 非公平的共享模式获取许可证
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        // 当前可用许可证数量
        int available = getState();
        /*
         * 尝试获取许可证，当前可用许可证数量小于等于0时，返回负值，表示获取失败，
         * 当前可用许可证大于0时才可能获取成功，CAS失败了会循环重新获取最新的值尝试获取
         */
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

调用`semaphore.release();` ，线程尝试释放许可证，并使用 CAS 操作去修改 `state` 的值 `state=state+1`。释放许可证成功之后，同时会唤醒同步队列中的一个线程。被唤醒的线程会重新尝试去修改 `state` 的值 `state=state-1` ，如果 `state>=0` 则获取令牌成功，否则重新进入阻塞队列，挂起线程。

```java
// 释放一个许可证
public void release() {
    sync.releaseShared(1);
}


// 释放一个或者多个许可证
public void release(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.releaseShared(permits);
}


// 释放共享锁，同时会唤醒同步队列中的一个线程。
public final boolean releaseShared(int arg) {
    //释放共享锁
    if (tryReleaseShared(arg)) {
      //唤醒同步队列中的一个线程
      doReleaseShared();
      return true;
    }
    return false;
}
```

`tryReleaseShared` 方法是`Semaphore` 的内部类 `Sync` 重写的一个方法

```java
// 内部类 Sync 中重写的一个方法
// 尝试释放资源
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        // 可用许可证+1
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
         // CAS修改state的值
        if (compareAndSetState(current, next))
            return true;
    }
}

```

#### **使用场景**

```java
public class SemaphoreExample {
  // 请求的数量
  private static final int threadCount = 550;

  public static void main(String[] args) throws InterruptedException {
    // 创建一个具有固定线程数量的线程池对象（如果这里线程池的线程数量给太少的话你会发现执行的很慢）
    ExecutorService threadPool = Executors.newFixedThreadPool(300);
    // 初始许可证数量
    final Semaphore semaphore = new Semaphore(20);

    for (int i = 0; i < threadCount; i++) {
      final int threadnum = i;
      threadPool.execute(() -> {// Lambda 表达式的运用
        try {
          semaphore.acquire();// 获取一个许可，所以可运行线程数量为20/1=20
          test(threadnum);
          semaphore.release();// 释放一个许可
        } catch (InterruptedException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        }

      });
    }
    threadPool.shutdown();
    System.out.println("finish");
  }

  public static void test(int threadnum) throws InterruptedException {
    Thread.sleep(1000);// 模拟请求的耗时操作
    System.out.println("threadnum:" + threadnum);
    Thread.sleep(1000);// 模拟请求的耗时操作
  }
}

```

执行 `acquire()` 方法阻塞，直到有一个许可证可以获得然后拿走一个许可证；每个 `release` 方法增加一个许可证，这可能会释放一个阻塞的 `acquire()` 方法。然而，其实并没有实际的许可证这个对象，`Semaphore` 只是维持了一个可获得许可证的数量。 `Semaphore` 经常用于限制获取某种资源的线程数量。

当然一次也可以一次拿取和释放多个许可，不过一般没有必要这样做：

```java
semaphore.acquire(5);// 获取5个许可，所以可运行线程数量为20/5=4
test(threadnum);
semaphore.release(5);// 释放5个许可
```

除了 `acquire()` 方法之外，另一个比较常用的与之对应的方法是 `tryAcquire()` 方法，该方法如果获取不到许可就立即返回 false。





###  **CountDownLatch**

`CountDownLatch` 允许 `count` 个线程阻塞在一个地方，直至所有线程的任务都执行完毕。

**`CountDownLatch` 是一次性的，**计数器的值只能在构造方法中初始化一次，之后没有任何机制再次对其设置值，当 `CountDownLatch` 使用完毕后，它不能再次被使用。

####  **原理**

`CountDownLatch` 是共享锁的一种实现,它默认构造 AQS 的 `state` 值为 `count`。当线程使用 `countDown()` 方法时,其实使用了`tryReleaseShared`方法以 CAS 的操作来减少 `state`,直至 `state` 为 0 。**当调用 `await()` 方法的时候，如果 `state` 不为 0，那就证明任务还没有执行完毕，`await()` 方法就会一直阻塞**，也就是说 `await()` 方法之后的语句不会被执行。直到`count` 个线程调用了`countDown()`使 state 值被减为 0，或者调用`await()`的线程被中断，该线程才会从阻塞中被唤醒，`await()` 方法之后的语句得到执行。

#### **使用场景**

`CountDownLatch` 的作用就是 允许 count 个线程阻塞在一个地方，直至所有线程的任务都执行完毕。之前在项目中，有一个使用多线程读取多个文件处理的场景，我用到了 `CountDownLatch` 。具体场景是下面这样的：

我们要读取处理 6 个文件，这 6 个任务都是没有执行顺序依赖的任务，但是我们需要返回给用户的时候将这几个文件的处理的结果进行统计整理。

为此我们定义了一个线程池和 count 为 6 的`CountDownLatch`对象 。使用线程池处理读取任务，每一个线程处理完之后就将 count-1，**调用`CountDownLatch`对象的 `await()`方法，直到所有文件读取完之后，才会接着执行后面的逻辑。**

```java
public class CountDownLatchExample1 {
    // 处理文件的数量
    private static final int threadCount = 6;

    public static void main(String[] args) throws InterruptedException {
        // 创建一个具有固定线程数量的线程池对象（推荐使用构造方法创建）
        ExecutorService threadPool = Executors.newFixedThreadPool(10);
        final CountDownLatch countDownLatch = new CountDownLatch(threadCount);
        for (int i = 0; i < threadCount; i++) {
            final int threadnum = i;
            threadPool.execute(() -> {
                try {
                    //处理文件的业务操作
                    //......
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    //表示一个文件已经被完成
                    countDownLatch.countDown();
                }

            });
        }
        countDownLatch.await(); //count不为0会被一直阻塞
        threadPool.shutdown();
        System.out.println("finish");
    }
}

```

**有没有可以改进的地方呢？**

可以使用 `CompletableFuture` 类来改进！Java8 的 `CompletableFuture` 提供了很多对多线程友好的方法，使用它可以很方便地为我们编写多线程程序，什么异步、串行、并行或者等待所有线程执行完任务什么的都非常方便。



### **CyclicBarrier**

`CyclicBarrier` 和 `CountDownLatch` 非常类似，它也可以实现线程间的技术等待，但是它的功能比 `CountDownLatch` 更加复杂和强大。主要应用场景和 `CountDownLatch` 类似。`CyclicBarrier` 的字面意思是**可循环使用（Cyclic）的屏障（Barrier）**。它要做的事情是：让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活。

`CountDownLatch` 的实现是基于 AQS 的，而 `CycliBarrier` 是基于 `ReentrantLock`(`ReentrantLock` 也属于 AQS 同步器)和 `Condition` 的。**是可重复使用的**，当所有线程都到达屏障点后，屏障会重置并再次等待所有线程到达。另外，`CyclicBarrier` 还提供一个更高级的构造函数 `CyclicBarrier(int parties, Runnable barrierAction)`，**用于在线程到达屏障时，优先执行 `barrierAction`**，方便处理更复杂的业务场景。





#### **原理**

`CyclicBarrier` 内部通过一个 `count` 变量作为计数器，`count` 的初始值为 `parties` 属性的初始化值，每当一个线程到了栅栏这里了，那么就将计数器减 1。如果 count 值为 0 了，表示这是这一代最后一个线程到达栅栏，就尝试执行我们构造方法中输入的任务。

```java
//每次拦截的线程数
private final int parties;
//计数器
private int count;
```

`CyclicBarrier` 默认的构造方法是 `CyclicBarrier(int parties)`，其参数表示屏障拦截的线程数量，每个线程调用 `await()` 方法告诉 `CyclicBarrier` 我已经到达了屏障，然后当前线程被阻塞。

```java
public CyclicBarrier(int parties) {
    this(parties, null);
}

public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```

其中，`parties` 就代表了有拦截的线程的数量，当拦截的线程数量达到这个值的时候就打开栅栏，让所有线程通过。

2、当调用 `CyclicBarrier` 对象调用 `await()` 方法时，实际上调用的是 `dowait(false, 0L)`方法。 `await()` 方法就像树立起一个栅栏的行为一样，将线程挡住了，当拦住的线程数量达到 `parties` 的值时，栅栏才会打开，线程才得以通过执行。

```java
public int await() throws InterruptedException, BrokenBarrierException {
  try {
      return dowait(false, 0L);
  } catch (TimeoutException toe) {
      throw new Error(toe); // cannot happen
  }
}

```



#### **使用场景**

```java
public class CyclicBarrierExample2 {
  // 请求的数量
  private static final int threadCount = 550;
  // 需要同步的线程数量
  private static final CyclicBarrier cyclicBarrier = new CyclicBarrier(5, () -> {
    System.out.println("------当线程数达到之后，优先执行------");
  });

  public static void main(String[] args) throws InterruptedException {
    // 创建线程池
    ExecutorService threadPool = Executors.newFixedThreadPool(10);

    for (int i = 0; i < threadCount; i++) {
      final int threadNum = i;
      Thread.sleep(1000);
      threadPool.execute(() -> {
        try {
          test(threadNum);
        } catch (InterruptedException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        } catch (BrokenBarrierException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        }
      });
    }
    threadPool.shutdown();
  }

  public static void test(int threadnum) throws InterruptedException, BrokenBarrierException {
    System.out.println("threadnum:" + threadnum + "is ready");
    cyclicBarrier.await();
    System.out.println("threadnum:" + threadnum + "is finish");
  }

}

```

运行结果

```java
threadnum:0is ready
threadnum:1is ready
threadnum:2is ready
threadnum:3is ready
threadnum:4is ready
------当线程数达到之后，优先执行------
threadnum:4is finish
threadnum:0is finish
threadnum:2is finish
threadnum:1is finish
threadnum:3is finish
threadnum:5is ready
threadnum:6is ready
threadnum:7is ready
threadnum:8is ready
threadnum:9is ready
------当线程数达到之后，优先执行------
threadnum:9is finish
threadnum:5is finish
threadnum:6is finish
threadnum:8is finish
threadnum:7is finish
......

```

