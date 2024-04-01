## AQS

AQS队列同步器,用来构建锁的基础框架,Lock实现类都是基于AQS实现的。

AQS是基于模板方法模式进行设计的,所以锁的实现需要继承AQS并重写它指定的方法。 

AQS内部定义了一个FIFO的队列来实现线程的同步,同时还定义了同步状态来记录锁的信息。 

AQS的模板方法,将管理同步状态的逻辑提炼出来形成标准流程,这些方法主要包括：独占式获取同步状态、独占式释放同步状态、共享式获取同步状态、共享式释放同步状态。

AQS 定义两种资源共享方式：`Exclusive`（独占，只有一个线程能执行，如`ReentrantLock`）和`Share`（共享，多个线程可同时执行，如`Semaphore`/`CountDownLatch`）。

一般来说，自定义同步器的共享方式要么是独占，要么是共享，他们也只需实现`tryAcquire-tryRelease`、`tryAcquireShared-tryReleaseShared`中的一种即可。但 AQS 也支持自定义同步器同时实现独占和共享两种方式，如`ReentrantReadWriteLock`。



### **AQS 的原理是什么？**

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

CLH 锁数据结构很简单，类似一个链表队列，所有请求获取锁的线程会排列在链表队列中，自旋访问队列中前一个节点的状态。**当一个节点释放锁时，只有它的后一个节点才可以得到锁。**CLH 锁本身有一个队尾指针 Tail，它是一个原子变量，指向队列最末端的 CLH 节点。每一个 CLH 节点有两个属性：所代表的线程和标识是否持有锁的状态变量。当一个线程要获取锁时，它会对 Tail 进行一个 getAndSet 的原子操作。该操作会返回 Tail 当前指向的节点，也就是当前队尾节点，然后使 Tail 指向这个线程对应的 CLH 节点，成为新的队尾节点。入队成功后，该线程会轮询上一个队尾节点的状态变量，当上一个节点释放锁后，它将得到这个锁。



#### **CLH 优缺点分析**

CLH 锁作为自旋锁的改进，有以下几个优点：

1. 性能优异，获取和释放锁开销小。CLH 的锁状态不再是单一的原子变量，而是分散在每个节点的状态中，降低了自旋锁在竞争激烈时频繁同步的开销。在释放锁的开销也因为不需要使用 CAS 指令而降低了。
2. 公平锁。先入队的线程会先得到锁。
3. 实现简单，易于理解。
4. 扩展性强。下面会提到 AQS 如何扩展 CLH 锁实现了JUC 包下各类丰富的同步器。

当然，它也有两个缺点：第一是因为有自旋操作，当锁持有时间长时会带来较大的 CPU 开销。第二是基本的 CLH 锁功能单一，不改造不能支持复杂的功能。

在原始版本的 CLH 锁中，节点间甚至都没有互相链接。但是，通过在节点中显式地维护前驱节点，CLH 锁就可以处理“超时”和各种形式的“取消”：如果一个节点的前驱节点取消了，这个节点就可以滑动去使用前面一个节点的状态字段。对于通过自旋获取锁的 CLH 锁来说，只需要显式的维护前驱节点就可以实现取消功能，如下图所示：

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240324222041328.png" alt="image-20240324222041328" style="zoom:50%;" />

#### **AQS 对 CLH 队列锁的改造**

针对 CLH 的缺点，AQS 对 CLH 队列锁进行了一定的改造。针对第一个缺点，**AQS 将自旋操作改为阻塞线程操作**。针对第二个缺点，AQS 对 CLH 锁进行改造和扩展，原作者 Doug Lea 称之为“CLH 锁的变体”。下面将详细讲 AQS 底层细节以及对 CLH 锁的改进。AQS 中的对 CLH 锁数据结构的改进主要包括三方面：**扩展每个节点的状态、显式的维护前驱节点和后继节点以及诸如出队节点显式设为 null** 等辅助 GC 的优化。正是这些改进使 AQS 可以支撑 j.u.c 丰富多彩的同步器实现。

**显示的添加了前后指针**

因为 AQS 用阻塞等待替换了自旋操作，线程会阻塞等待锁的释放，不能主动感知到前驱节点状态变化的信息。AQS 中显式的维护前驱节点和后继节点，**需要释放锁的节点会显式通知下一个节点解除阻塞**，如下图所示，T1 释放锁后主动唤醒 T2，使 T2 检测到锁已释放，获取锁成功。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240324222229937.png" alt="image-20240324222229937" style="zoom:50%;" />



 **扩展每个节点的状态**

AQS 每个节点的状态如下所示，在源码中如下所示：

```
volatile int waitStatus;
```

AQS 同样提供了该状态变量的原子读写操作，但和同步器状态不同的是，节点状态在 AQS 中被清晰的定义，如下表所示

| 状态名    | 枚举值 | 描述                                     |
| --------- | ------ | ---------------------------------------- |
| SIGNAL    | -1     | 表示该节点正常等待                       |
| PROPAGATE | -3     | 应将 releaseShared 传播到其他节点        |
| CONDITION | -2     | 该节点位于条件队列，不能用于同步队列节点 |
| CANCELLED | 1      | 由于超时、中断或其他原因，该节点被取消   |
| 0         | 0      | 结点初始默认值或已经被释放               |



#### **独占模式分析**

head节点代表当前正在持有锁的节点，如果当前线程所在的节点处于头节点的后一个，那么它将会不断去尝试拿锁，直到获取成功。否则进行判断，是否需要挂起。这样就能保证head之后的一个节点在自旋CAS获取锁，其他线程都已经被挂起或正在被挂起。这样就能最大限度地避免无用的自旋消耗CPU。



**请求锁**

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240324223401016.png" alt="image-20240324223401016" style="zoom: 67%;" />

**将线程加入队列**

先来看addWaiter，当线程获取锁失败时，会调用addWaiter方法加入到队列中，并将tail指向该结点，这是一个原子性的操作

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240324223827728.png" alt="image-20240324223827728" style="zoom: 67%;" />

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240324224514923.png" alt="image-20240324224514923" style="zoom: 80%;" />

如果队列未初始化(tail== null)，那么就尝试初始化，如果尾插节点失败，那么就不断重试，直到插入成功为止。

**判断能否获取锁**

再来看acquireQueued方法

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240324224425501.png" alt="image-20240324224425501" style="zoom: 67%;" />

当前置节点为head，说明当前节点有权限去尝试拿锁，这是一种约定。如果tryAcquire返回true，代表拿到了锁，那么顺理成章，函数返回。**如果tryAcquire为false，由于死循环，会一直执行tryAcquire，也就是会自旋等待，直到拿到锁为止**

若当前节点没有拿锁的权限或拿锁失败，那么将会进入shouldParkAfterFailedAcquire判断是否需要挂起(park)，方法的参数是pred Node和当前Node的引用。**若需要挂起，interrupted将会设置为ture，终止死循环**

**判断是否需要挂起**

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240324220735021.png" alt="image-20240324220735021" style="zoom:50%;" />

若pred的waitSatus为SIGNAL，说明前置节点也在等待拿锁，所以当前线程可以挂起休息，返回true，之后**会调用parkAndCheckInterrupt将该线程挂起**

如果ws大于0，说明pred的waitSatus是CANCEL，所以可以将其从队列中删除。**这里通过从后向前搜索**，将pred指向搜索过程中第一个waitSatus为非CANCEL的节点。相当于链式地删除被CANCEL的节点。然后返回false，代表当前节点不需要挂起，**因为pred指向了新的Node，需要重试外层的逻辑。**

除此之外，pred的ws还有两种可能，0或PROPAGATE，有人可能会问，为什么不可能是CONDITION？因为waitStatus只有在其他条件模式下，才会被修改为CONDITION，这里不会出现，并且只有在共享模式下，才可能出现waitStatus为PROPAGATE，暂时也不用管。

那么在独占模式下，ws在这里只会出现0的情况。0代表pred处于初始化默认状态，所以通过CAS将当前pred的waitStatus修改为SIGNAL，然后返回false，重试外层逻辑。

**释放锁**

主要是通过unparkSuccessor来唤醒后续线程

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240324230613040.png" alt="image-20240324230613040" style="zoom: 67%;" />



**唤醒后续线程**

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240324230827242.png" alt="image-20240324230827242" style="zoom:50%;" />

获取head的waitStatus，如果不为0，那么将其置为0，表示锁已释放。

接下来获取后续节点如果后续节点为null或者处于CANCELED状态，那么从后往前搜索，**找到除了head外最靠前且非CANCELED状态的Node，对其进行唤醒，让它起来尝试拿锁。**





### **自定义同步器**

同步器的设计是基于模板方法模式的，如果需要自定义同步器一般的方式是这样（模板方法模式很经典的一个应用）：

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

`Semaphore` 是共享锁的一种实现，它默认构造 AQS 的 `state` 值为 `permits`，你可以将 `permits` 的值理解为许可证的数量，只有拿到许可证的线程才能执行。

调用`semaphore.acquire()` ，线程尝试获取许可证，如果 `state >= 0` 的话，则表示可以获取成功。如果获取成功的话，使用 CAS 操作去修改 `state` 的值 `state=state-1`。如果 `state<0` 的话，则表示许可证数量不足。此时会创建一个 Node 节点加入阻塞队列，挂起当前线程。

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

实战

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









