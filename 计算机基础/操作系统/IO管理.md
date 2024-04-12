

## DMA 技术

在没有 DMA 技术前，I/O 的过程是这样的：

- CPU 发出对应的指令给磁盘控制器，然后返回；
- 磁盘控制器收到指令后，于是就开始准备数据，会把数据放入到磁盘控制器的内部缓冲区中，然后产生一个**中断**；
- CPU 收到中断信号后，停下手头的工作，接着把磁盘控制器的缓冲区的数据一次一个字节地读进自己的寄存器，然后再把寄存器里的数据写入到内存，而在数据传输的期间 CPU 是无法执行其他任务的。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240408113434460.png" alt="image-20240408113434460" style="zoom: 67%;" />

可以看到，整个数据的传输过程，都要需要 CPU 亲自参与搬运数据的过程，而且这个过程，CPU 是不能做其他事情的。



**DMA**

DMA 技术，也就是**直接内存访问（Direct Memory Access）** 技术。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240408113626155.png" alt="image-20240408113626155" style="zoom: 67%;" />

什么是 DMA 技术？简单理解就是，**在进行 I/O 设备和内存的数据传输的时候，数据搬运的工作全部交给 DMA 控制器，而 CPU 不再参与任何与数据搬运相关的事情，这样 CPU 就可以去处理别的事务**。

具体过程：

- 用户进程调用 read 方法，向操作系统发出 I/O 请求，请求读取数据到自己的内存缓冲区中，进程进入阻塞状态；
- 操作系统收到请求后，进一步**将 I/O 请求发送 DMA**，然后让 CPU 执行其他任务；
- DMA 进一步将 I/O 请求发送给磁盘；
- 磁盘收到 DMA 的 I/O 请求，把数据从磁盘读取到磁盘控制器的缓冲区中，当磁盘控制器的缓冲区被读满后，向 DMA 发起中断信号，告知自己缓冲区已满；
- **DMA 收到磁盘的信号，将磁盘控制器缓冲区中的数据拷贝到内核缓冲区中，此时不占用 CPU，CPU 可以执行其他任务**；
- 当 DMA 读取了足够多的数据，就会**发送中断信号给 CPU**；
- CPU 收到 DMA 的信号，知道数据已经准备好，于是将数据从内核拷贝到用户空间，系统调用返回；

可以看到， **CPU 不再参与「将数据从磁盘控制器缓冲区搬运到内核空间」的工作，这部分工作全程由 DMA 完成**。CPU 在这个过程中只需要告诉DMA传输什么数据，从哪里传输到哪里





## **NIO 零拷贝**



### **传统的网络文件传输的问题？**

如果服务端要提供文件传输的功能，我们能想到的最简单的方式是：将磁盘上的文件读取出来，然后通过网络协议发送给客户端。

传统 I/O 的工作方式是，**数据读取和写入是从用户空间到内核空间来回复制**，而内核空间的数据是通过操作系统层面的 I/O 接口从磁盘读取或写入。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240408114051656.png" alt="image-20240408114051656" style="zoom:50%;" />

首先，期间共**发生了 4 次用户态与内核态的上下文切换**，因为发生了两次系统调用，一次是 `read()` ，一次是 `write()`，每次系统调用都得先从用户态切换到内核态，等内核完成任务后，再从内核态切换回用户态。

上下文切换到成本并不小，一次切换需要耗时几十纳秒到几微秒，虽然时间看上去很短，但是在高并发的场景下，这类时间容易被累积和放大，从而影响系统的性能。

其次，还**发生了 4 次数据拷贝**，其中两次是 DMA 的拷贝，另外两次则是通过 CPU 拷贝的

所以，**要想提高文件传输的性能，就需要减少「用户态与内核态的上下文切换」和「内存拷贝」的次数**。





###  **如何实现零拷贝？**

**要想提高文件传输的性能，就需要减少「用户态与内核态的上下文切换」和「内存拷贝」的次数**。

零拷贝技术实现的方式通常有 2 种：

- mmap + write 
- sendfile



#### **mmap + write**

**减少了1次数据拷贝（通过映射的方式较少了内核缓冲区到用户缓冲区的数据拷贝），系统调用依然是2次，有4次上下文切换**

在前面我们知道，`read()` 系统调用的过程中会把内核缓冲区的数据拷贝到用户的缓冲区里，于是为了减少这一步开销，我们可以用 `mmap()` 替换 `read()` 系统调用函数。

```c
buf = mmap(file, len);
write(sockfd, buf, len);
```

`mmap()` 系统调用函数会直接把内核缓冲区里的数据「**映射**」到用户空间，这样，操作系统内核与用户空间就不需要再进行任何的数据拷贝操作。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240408114733378.png" alt="image-20240408114733378" style="zoom:50%;" />

具体过程如下：

- 应用进程调用了 `mmap()` 后，DMA 会把磁盘的数据拷贝到内核的缓冲区里。接着，应用进程跟操作系统内核「共享」这个缓冲区；
- 应用进程再调用 `write()`，操作系统直接将内核缓冲区的数据拷贝到 socket 缓冲区中，这一切都发生在内核态，由 CPU 来搬运数据；
- 最后，把内核的 socket 缓冲区里的数据，拷贝到网卡的缓冲区里，这个过程是由 DMA 搬运的。

我们可以得知，通过使用 `mmap()` 来代替 `read()`， 可以减少一次数据拷贝的过程。

但这还不是最理想的零拷贝，因为仍然需要通过 CPU 把内核缓冲区的数据拷贝到 socket 缓冲区里，**而且仍然需要 4 次上下文切换，因为系统调用还是 2 次。**



#### **sendfile**

**减少了一次系统调用（两次上下文切换），由sendfile代替read和write，数据拷贝减少1次，数据直接由内核缓冲区传送到socket缓冲区。**

在 Linux 内核版本 2.1 中，提供了一个专门发送文件的系统调用函数 `sendfile()`，函数形式如下：

```c
#include <sys/socket.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

它的前两个参数分别是目的端和源端的文件描述符，后面两个参数是源端的偏移量和复制数据的长度，返回值是实际复制数据的长度。

首先，它可以替代前面的 `read()` 和 `write()` 这两个系统调用，这样就可以减少一次系统调用，也就减少了 2 次上下文切换的开销。

其次，该系统调用，可以直接把内核缓冲区里的数据拷贝到 socket 缓冲区里，不再拷贝到用户态，这样就只有 2 次上下文切换，和 3 次数据拷贝。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240408115538267.png" alt="image-20240408115538267" style="zoom: 67%;" />



#### **sendfile + SG-DMA**

**SG-DMA直接将内核缓冲区数据复制到网卡，减少了2次数据拷贝，总共只需要1次系统调用（2次上下文切换），2次的数据拷贝（CPU不参与）**

如果网卡支持 SG-DMA（*The Scatter-Gather Direct Memory Access*）技术（和普通的 DMA 有所不同），我们可以进一步减少通过 CPU 把内核缓冲区里的数据拷贝到 socket 缓冲区的过程。

你可以在你的 Linux 系统通过下面这个命令，查看网卡是否支持 scatter-gather 特性：

```bash
$ ethtool -k eth0 | grep scatter-gather
scatter-gather: on
```

于是，从 Linux 内核 `2.4` 版本开始起，对于支持网卡支持 SG-DMA 技术的情况下， `sendfile()` 系统调用的过程发生了点变化，具体过程如下：

- 第一步，通过 DMA 将磁盘上的数据拷贝到内核缓冲区里；
- 第二步，缓冲区描述符和数据长度传到 socket 缓冲区，这样网卡的 SG-DMA 控制器就可以直接将内核缓存中的数据拷贝到网卡的缓冲区里，此过程不需要将数据从操作系统内核缓冲区拷贝到 socket 缓冲区中，这样就减少了一次数据拷贝；

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240408120142851.png" alt="image-20240408120142851" style="zoom:50%;" />

这就是所谓的**零拷贝（Zero-copy）技术，因为我们没有在内存层面去拷贝数据，也就是说全程没有通过 CPU 来搬运数据，所有的数据都是通过 DMA 来进行传输的。**。

零拷贝技术的文件传输方式相比传统文件传输的方式，减少了 2 次上下文切换和数据拷贝次数，**只需要 2 次上下文切换和数据拷贝次数，就可以完成文件的传输，而且 2 次的数据拷贝过程，都不需要通过 CPU，2 次都是由 DMA 来搬运。**

所以，总体来看，**零拷贝技术可以把文件传输的性能提高至少一倍以上**。



#### **总结**

零拷贝是提升 IO 操作性能的一个常用手段，像 ActiveMQ、Kafka 、RocketMQ、QMQ、Netty 等顶级开源项目都用到了零拷贝。

零拷贝是指计算机执行 IO 操作时，**CPU 不需要将数据从一个存储区域复制到另一个存储区域**，从而可以**减少上下文切换以及 CPU 的拷贝时间**。也就是说，零拷贝主**主要解决操作系统在处理 I/O 操作时频繁复制数据的问题**。零拷贝的常见实现技术有： `mmap+write`、`sendfile`和 `sendfile + DMA gather copy` 。

下图展示了各种零拷贝技术的对比图：

|                         | CPU 拷贝 | DMA 拷贝 | 系统调用   | 上下文切换 |
| ----------------------- | -------- | -------- | ---------- | ---------- |
| 传统方法                | 2        | 2        | read+write | 4          |
| mmap+write              | 1        | 2        | mmap+write | 4          |
| sendfile                | 1        | 2        | sendfile   | 2          |
| sendfile + SG-DMA  copy | 0        | 2        | sendfile   | 2          |

可以看出，无论是传统的 I/O 方式，还是引入了零拷贝之后，2 次 DMA(Direct Memory Access) 拷贝是都少不了的。因为两次 DMA 都是依赖硬件完成的。**零拷贝主要是减少了 CPU 拷贝及上下文的切换。**

**Java 对零拷贝的支持：**

- `MappedByteBuffer` 是 NIO 基于内存映射（`mmap`）这种零拷⻉⽅式的提供的⼀种实现，底层实际是调用了 Linux 内核的 **`mmap` 系统调用**。它可以将一个文件或者文件的一部分映射到内存中，形成一个**虚拟内存文件**，这样就可以直接操作内存中的数据，而不需要通过系统调用来读写文件。
- `FileChannel` 的`transferTo()/transferFrom()`是 NIO 基于发送文件（`sendfile`）这种零拷贝方式的提供的一种实现，底层实际是调用了 Linux 内核的 `sendfile`系统调用。它可以直接将文件数据从磁盘发送到网络，而不需要经过用户空间的缓冲区。关于`FileChannel`的用法可以看看这篇文章：[Java NIO 文件通道 FileChannel 用法open in new window](https://www.cnblogs.com/robothy/p/14235598.html)。











## **IO模型**

### **BIO (Blocking I/O)**

**BIO 属于同步阻塞 IO 模型** 。

同步阻塞 IO 模型中，应用程序发起 read 调用后，会一直阻塞，直到内核把数据拷贝到用户空间。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240407095054091.png" alt="image-20240407095054091" style="zoom: 67%;" /> 

在客户端连接数量不高的情况下，是没问题的。但是，当面对十万甚至百万级连接的时候，传统的 BIO 模型是无能为力的。因此，我们需要一种更高效的 I/O 处理模型来应对更高的并发量。



### **NIO (Non-blocking/New I/O)**

Java 中的 NIO 于 Java 1.4 中引入，对应 `java.nio` 包，提供了 `Channel` , `Selector`，`Buffer` 等抽象。NIO 中的 N 可以理解为 Non-blocking，不单纯是 New。它是支持面向缓冲的，基于通道的 I/O 操作方法。 对于高负载、高并发的（网络）应用，应使用 NIO 。它在标准 Java 代码中提供了非阻塞、面向缓冲、基于通道的 I/O，可以使用**少量的线程来处理多个连接**，大大提高了 I/O 效率和并发。

Java 中的 NIO 可以看作是 **I/O 多路复用模型**。也有很多人认为，Java 中的 NIO 属于同步非阻塞 IO 模型。

我们先来看看 **同步非阻塞 IO 模型**。

同步非阻塞 IO 模型中，应用程序会一直发起 read 调用，等待数据就绪，如果就绪了数据从内核空间拷贝到用户空间的这段时间里，线程依然是阻塞的，直到在内核把数据拷贝到用户空间。

相比于同步阻塞 IO 模型，同步非阻塞 IO 模型确实有了很大改进**。通过轮询操作，避免了一直阻塞，如果内核态数据未就绪，就一直返回-1**

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240407095241608.png" style="zoom: 67%;" />

但是，这种 IO 模型同样存在问题：**应用程序不断进行 I/O 系统调用轮询数据是否已经准备好的过程是十分消耗 CPU 资源的。**这个时候，**I/O 多路复用模型** 就上场了。

需要注意：使用 NIO 并不一定意味着高性能，它的性能优势主要体现在高并发和高延迟的网络环境下。当连接数较少、并发程度较低或者网络传输速度较快时，NIO 的性能并不一定优于传统的 BIO 。





### **I/O 多路复用模型**

I/O 多路复用是一种使得程序能同时监听多个文件描述符的技术,从而提高程序的性能。I/O 多路复用能够在单个线程中，通过监视多个 I/O 流的状态来同时管理多个 I/O 流，一旦检测到某个文件描述符上我们关心的事件发生（就绪）,能够通知程序进行相应的处理（读写操作）。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240310104140781.png" alt="image-20240310104140781" style="zoom: 50%;" />



IO 多路复用模型中，线程首先发起 select 调用，询问内核数据是否准备就绪，等内核把数据准备好了，用户线程再发起 read 调用。read 调用的过程（数据从内核空间 -> 用户空间）还是阻塞的。

> 目前支持 IO 多路复用的系统调用，有 select，epoll 等等。select 系统调用，目前几乎在所有的操作系统上都有支持。
>
> - **select 调用**：内核提供的系统调用，它支持一次查询多个系统调用的可用状态。几乎所有的操作系统都支持。
> - **epoll 调用**：linux 2.6 内核，属于 select 调用的增强版本，优化了 IO 的执行效率

**IO 多路复用模型，通过减少无效的系统调用，减少了对 CPU 资源的消耗。**

Java 中的 NIO ，有一个非常重要的**选择器 ( Selector )** 的概念，也可以被称为 **多路复用器**。通过它，只需要一个线程便可以管理多个客户端连接。当客户端数据到了之后，才会为其服务。





#### **select**

首先要构造一个关于文件描述符的列表，将要监听的文件描述符添加到该列表中，这个文件描述符的列表数据类型为 fd_set，它是一个**整型数组，总共是 1024 个比特位**，每一个比特位代表一个文件描述符的状态。比如当需要 select 检测时，这一位为 0 就表示不检测对应的文件描述符的事件,为 1 表示检测对应的文件描述符的事件。 

主要分三步：

1. 调用 select() 系统调用，将fdlist复制到内核态，这个函数是阻塞的
2. 内核态对fdlist进行遍历，直到这些描述符中的一个或者多个进行 I/O 操作时（channel对象绑定的内核缓冲区有数据到达），该函数才返回，并修改文件描述符的列表中对应的值，0 表示没有检测到该事件，1 表示检测到该事件。
3. select() 返回时。会告诉进程有多少描述符要进行 I/O 操作。接下来用户态遍历文件描述符的列表进行 I/O 操作（read或write）。 

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240407155306459.png" alt="image-20240407155306459" style="zoom: 67%;" />



**缺点**

- 每次调用select都需要把 fd 集合从用户态拷贝到内核态，这个开销在 fd 很多时会很大； 

- 同时每次调用 select 都需要在内核遍历传递进来的所有 fd，这个开销在 fd 很多时也很大；

- **select 支持的文件描述符数量太小了**，默认是 1024（32位）/2048（64位）；

- 文件描述符集合不能重用，因为内核每次检测到事件都会修改，所以每次都需要重置； 

- 每次 select 返回后，只能知道有几个 fd 发生了事件，但是具体哪几个还需要遍历文件描述符集合进一步判断。 




#### **poll**

poll的实现和select非常相似，只是描述fd集合的方式不同。poll只是使用pollfd结构而不是select的fd_set结构，这就解决了select的fds集合大小1024限制问题。

但poll和select同样存在一个性能缺点就是包含大量文件描述符的数组被整体复制于用户态和内核的地址空间之间，而不论这些文件描述符是否就绪，它的开销随着文件描述符数量的增加而线性增大。



#### **epoll**

epoll 是一种更加高效的 IO 复用技术，epoll 的使用步骤及原理如下： 

调用 epoll_create() 会在内核中创建一个 eventpoll 结构体数据，称之为 epoll 对象，在这个结构体中有 2 个比较重要的数据成员，**一个是需要检测的文件描述符的信息 struct_root rbr（红黑树）**；还有一个是就绪列表struct list_head rdlist，存放检测到数据到达的文件描述符信息（双向链表）。

`epoll` 使用了**事件驱动（回调方式）**的方式，可以同时监视大量的文件描述符，并且不随文件描述符数量增加而降低效率。它通过系统调用 `epoll_ctl()` 来注册和修改需要监视的文件描述符及其事件类型，然后通过 `epoll_wait()` 等待事件的发生，将就绪的事件存放在就绪队列中。**这种基于事件的机制避免了传统的轮询方式，能够更快速地响应事件并处理多个并发连接。**，然后`epoll_wait()`返回后，将数据传送到用户态做进一步处理

![image-20240407171201161](https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240407171201161.png)



**epoll 的两种工作模式：** 

LT 模式（水平触发） LT（Level - Triggered）是缺省的工作方式，并且同时支持 Block 和 Nonblock Socket。在这种做法中，内核检测到一个文件描述符就绪了，就会通知用户对就绪的 fd 进行 IO 操作，**如果这个数据不被取走，内核就会一直通知。** 

ET 模式（边沿触发） ET（Edge - Triggered）是高速工作方式，只支持 Nonblock socket。在这种模式下，当描述符从未就绪变为就绪时，**内核通过 epoll 检测到，只会发送一次通知。**但是请注意，如果一直不对这个 fd 进行 IO 操作（从而导致它再次变成未就绪），内核不会发送更多的通知（only once）。 ET 模式在很大程度上减少了 epoll 事件被重复触发的次数，因此效率要比 LT 模式高。epoll 工作在 ET 模式的时候，必须使用非阻塞套接口，以避免由于一个文件描述符的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。

**优点**

- `epoll` 是 Linux 系统特有的一种高性能 I/O 多路复用机制。

- `epoll` **使用了事件驱动的方式，可以同时监视大量的文件描述符**，并且不随文件描述符数量增加而降低效率。

- `epoll` 提供了三个系统调用：`epoll_create()` 创建 `epoll` 实例，`epoll_ctl()` 添加或修改文件描述符的监听事件，`epoll_wait()` 等待文件描述符的事件就绪。

- `epoll` 使用了红黑树和双向链表等数据结构，能够快速有效地处理大量的并发连接。





#### **select和epoll的区别**

首先，从I/O模型的角度来看，select使用轮询模型。这意味着无论文件描述符是否活跃，select都会遍历所有的文件描述符集合。这种方式在处理大量文件描述符时效率较低，因为每次都需要检查所有的文件描述符。相比之下，epoll采用基于事件驱动的模型。它只会对活跃的文件描述符进行操作，**通过回调函数进行处理，从而大大提高了效率。**

其次，两者的文件描述符数量限制也不同。对于select，**其文件描述符数量被限制在1024左右**。这在处理大量并发连接时可能会成为问题，需要使用fd set多次扩展或者使用其他的多路复用技术。然而，epoll则没有这样的限制，它可以支持数万个文件描述符，因此非常适合处理高并发场景。

此外，两者在触发方式上也存在差异。select和epoll对可读事件和可写事件的触发方式并不相同，具体细节会根据具体的使用场景和编程需求有所不同。

总体而言，**`epoll` 在处理大量并发连接时通常比 `select` 更有效率**，特别是在 Linux 等支持 `epoll` 的系统上。在实际应用中，选择使用 `select` 还是 `epoll` 取决于应用的特定需求以及运行的操作系统。





#### **epoll为什么比select更高效？**

- epoll只会在调用epoll_ctl方法时才会将数据从用户态拷贝到内核态，每次调用select都需要把 fd 集合从用户态拷贝到内核态，这个开销在 fd 很多时会很大。
- epoll通过回调机制来通知已就绪的事件，而select通过轮询的方式来检测文件描述符是否就绪

- epoll调用epoll_wait时只会调用已就绪的事件，而select会把监听所有事件的文件描述符从内核态拷贝到用户态



#### **epoll一定比poll好吗？**

不一定

- `epoll` 在处理大量并发连接时表现更好，因为它的事件管理和通知机制更为高效。
- `poll` 在文件描述符数量较少时可能表现得更好，因为只有十几个请求时遍历结构体的开销也不大，但在大规模应用中可能受到性能影响。







### **Java中的NIO**

NIO 主要包括以下三个核心组件：

- **Buffer（缓冲区）**：NIO 读写数据都是通过缓冲区进行操作的。读操作的时候将 Channel 中的数据填充到 Buffer 中，而写操作时将 Buffer 中的数据写入到 Channel 中。
- **Channel（通道）**：Channel 是一个双向的、可读可写的数据传输通道，NIO 通过 Channel 来实现数据的输入输出。通道是一个抽象的概念，**它可以代表文件、套接字或者其他数据源之间的连接**。
- **Selector（选择器）**：**允许一个线程处理多个 Channel**，基于事件驱动的 I/O 多路复用模型。所有的 Channel 都可以注册到 Selector 上，由 Selector 来分配线程来处理事件。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240319111310279.png" alt="image-20240319111310279" style="zoom: 50%;" />

#### **Buffer（缓冲区）**

在传统的 BIO 中，数据的读写是面向流的， 分为字节流和字符流。

在 Java 1.4 的 NIO 库中，所有数据都是用缓冲区处理的，这是新库和之前的 BIO 的一个重要区别，有点类似于 BIO 中的缓冲流。NIO 在读取数据时，它是直接读到缓冲区中的。在写入数据时，写入到缓冲区中。 **使用 NIO 在读写数据时，都是通过缓冲区进行操作。**

`Buffer` 的子类如下图所示。其中，最常用的是 `ByteBuffer`，它可以用来存储和操作字节数据。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240324235849274.png" alt="image-20240324235849274" style="zoom:50%;" />

你可以将 Buffer 理解为一个数组，`IntBuffer`、`FloatBuffer`、`CharBuffer` 等分别对应 `int[]`、`float[]`、`char[]` 等。

为了更清晰地认识缓冲区，我们来简单看看`Buffer` 类中定义的四个成员变量：

```java
public abstract class Buffer {
    // Invariants: mark <= position <= limit <= capacity
    private int mark = -1;
    private int position = 0;
    private int limit;
    private int capacity;
}
```

这四个成员变量的具体含义如下：

1. 容量（`capacity`）：`Buffer`可以存储的最大数据量，`Buffer`创建时设置且不可改变；
2. 界限（`limit`）：`Buffer` 中可以读/写数据的边界。写模式下，`limit` 代表最多能写入的数据，一般等于 `capacity`（可以通过`limit(int newLimit)`方法设置）；读模式下，`limit` 等于 Buffer 中实际写入的数据大小。
3. 位置（`position`）：下一个可以被读写的数据的位置（索引）。从写操作模式到读操作模式切换的时候（flip），`position` 都会归零，这样就可以从头开始读写了。
4. 标记（`mark`）：`Buffer`允许将位置直接定位到该标记处，这是一个可选属性；

并且，上述变量满足如下的关系：**0 <= mark <= position <= limit <= capacity** 。

另外，Buffer 有读模式和写模式这两种模式，分别用于从 Buffer 中读取数据或者向 Buffer 中写入数据。Buffer 被创建之后默认是写模式，调用 `flip()` 可以切换到读模式。如果要再次切换回写模式，可以调用 `clear()` 或者 `compact()` 方法。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240325000046805.png" alt="image-20240325000046805" style="zoom: 50%;" />



#### **Channel（通道）**

Channel 是一个通道，它建立了与数据源（如文件、网络套接字等）之间的连接。我们可以利用它来读取和写入数据，就像打开了一条自来水管，让数据在 Channel 中自由流动。

BIO 中的流是单向的，分为各种 `InputStream`（输入流）和 `OutputStream`（输出流），数据只是在一个方向上传输。通道与流的不同之处在于通道是双向的，它可以用于读、写或者同时用于读写。

Channel 与前面介绍的 Buffer 打交道，读操作的时候将 Channel 中的数据填充到 Buffer 中，而写操作时将 Buffer 中的数据写入到 Channel 中。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240325000154821.png" alt="image-20240325000154821" style="zoom:50%;" />

另外，因为 Channel 是全双工的，所以它可以比流更好地映射底层操作系统的 API。特别是在 UNIX 网络编程模型中，底层操作系统的通道都是全双工的，同时支持读写操作。

其中，最常用的是以下几种类型的通道：

- `FileChannel`：文件访问通道；
- `SocketChannel`、`ServerSocketChannel`：TCP 通信通道；
- `DatagramChannel`：UDP 通信通道;

Channel 最核心的两个方法：

1. `read` ：读取数据并写入到 Buffer 中。
2. `write` ：将 Buffer 中的数据写入到 Channel 中。

这里我们以 `FileChannel` 为例演示一下是读取文件数据的。

```java
RandomAccessFile reader = new RandomAccessFile("/Users/guide/Documents/test_read.in", "r"))
FileChannel channel = reader.getChannel();
ByteBuffer buffer = ByteBuffer.allocate(1024);
channel.read(buffer);
```



#### **Selector（选择器）**

Selector（选择器） 是 NIO 中的一个关键组件，它允许一个线程处理多个 Channel。Selector 是基于事件驱动的 I/O 多路复用模型，主要运作原理是：通过 Selector 注册通道的事件，Selector 会不断地轮询注册在其上的 Channel。当事件发生时，比如：某个 Channel 上面有新的 TCP 连接接入、读和写事件，这个 Channel 就处于就绪状态，会被 Selector 轮询出来。Selector 会将相关的 Channel 加入到就绪集合中。通过 SelectionKey 可以获取就绪 Channel 的集合，然后对这些就绪的 Channel 进行相应的 I/O 操作。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240325003101227.png" alt="image-20240325003101227" style="zoom: 67%;" />

一个多路复用器 Selector 可以同时轮询多个 Channel，**由于 JDK 使用了 `epoll()` 代替传统的 `select` 实现**，所以它并没有最大连接句柄 `1024/2048` 的限制。这也就意味着只需要一个线程负责 Selector 的轮询，就可以接入成千上万的客户端。







### **AIO (Asynchronous I/O)**

AIO 也就是 NIO 2。Java 7 中引入了 NIO 的改进版 NIO 2,它是异步 IO 模型。

异步 IO 是基于事件和回调机制实现的，也就是应用操作之后会直接返回，不会堵塞在那里，**当后台处理完成，操作系统会通知相应的线程进行后续的操作。**

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240310104743965.png" alt="image-20240310104743965" style="zoom: 33%;" />



目前来说 AIO 的应用还不是很广泛。Netty 之前也尝试使用过 AIO，不过又放弃了。这是因为，Netty 使用了 AIO 之后，在 Linux 系统上的性能并没有多少提升。















