## RabbitMQ

### **RabbitMQ 是什么？**

RabbitMQ 是一个在 AMQP（Advanced Message Queuing Protocol ）基础上实现的，可复用的企业消息系统。它可以用于大型软件系统各个模块之间的高效通信，支持高并发，支持可扩展。它支持多种客户端如：Python、Ruby、.NET、Java、JMS、C、PHP、ActionScript、XMPP、STOMP 等，支持 AJAX，持久化，用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。

RabbitMQ 是使用 Erlang 编写的一个开源的消息队列，本身支持很多的协议：AMQP，XMPP, SMTP, STOMP，也正是如此，使的它变的非常重量级，更适合于企业级的开发。它同时实现了一个 Broker 构架，这意味着消息在发送给客户端时先在中心队列排队，对路由(Routing)、负载均衡(Load balance)或者数据持久化都有很好的支持。

PS:也可能直接问什么是消息队列？消息队列就是一个使用队列来通信的组件



### **AMQP 是什么?**

RabbitMQ 就是 AMQP 协议的 `Erlang` 的实现(当然 RabbitMQ 还支持 `STOMP2`、 `MQTT3` 等协议 ) AMQP 的模型架构 和 RabbitMQ 的模型架构是一样的，生产者将消息发送给交换器，交换器和队列绑定 。

RabbitMQ 中的交换器、交换器类型、队列、绑定、路由键等都是遵循的 AMQP 协议中相 应的概念。目前 RabbitMQ 最新版本默认支持的是 AMQP 0-9-1。

**AMQP 协议的三层**：

- **Module Layer**:协议最高层，主要定义了一些客户端调用的命令，客户端可以用这些命令实现自己的业务逻辑。
- **Session Layer**:中间层，主要负责客户端命令发送给服务器，再将服务端应答返回客户端，提供可靠性同步机制和错误处理。
- **TransportLayer**:最底层，主要传输二进制数据流，提供帧的处理、信道复用、错误检测和数据表示等。

**AMQP 模型的三大组件**：

- **交换器 (Exchange)**：消息代理服务器中用于把消息路由到队列的组件。
- **队列 (Queue)**：用来存储消息的数据结构，位于硬盘或内存中。
- **绑定 (Binding)**：一套规则，告知交换器消息应该将消息投递给哪个队列。



### **RabbitMQ 特点?**

- **可靠性**: RabbitMQ 使用一些机制来保证可靠性， 如持久化、传输确认及发布确认等。
- **灵活的路由** : 在消息进入队列之前，通过交换器来路由消息。对于典型的路由功能， RabbitMQ 己经提供了一些内置的交换器来实现。针对更复杂的路由功能，可以将多个交换器绑定在一起， 也可以通过插件机制来实现自己的交换器。
- **扩展性**: 多个 RabbitMQ 节点可以组成一个集群，也可以根据实际业务情况动态地扩展 集群中节点。
- **高可用性** : 队列可以在集群中的机器上设置镜像，使得在部分节点出现问题的情况下队 列仍然可用。
- **多种协议**: RabbitMQ 除了原生支持 AMQP 协议，还支持 STOMP， MQTT 等多种消息 中间件协议。
- **多语言客户端** :RabbitMQ 几乎支持所有常用语言，比如 Java、 Python、 Ruby、 PHP、 C#、 JavaScript 等。
- **管理界面** : RabbitMQ 提供了一个易用的用户界面，使得用户可以监控和管理消息、集 群中的节点等。
- **插件机制** : RabbitMQ 提供了许多插件 ， 以实现从多方面进行扩展，当然也可以编写自 己的插件。





### **RabbitMQ 核心概念？**

RabbitMQ 整体上是一个生产者与消费者模型，主要负责接收、存储和转发消息。可以把消息传递的过程想象成：当你将一个包裹送到邮局，邮局会暂存并最终将邮件通过邮递员送到收件人的手上，RabbitMQ 就好比由邮局、邮箱和邮递员组成的一个系统。从计算机术语层面来说，RabbitMQ 模型更像是一种交换机模型。

RabbitMQ 的整体模型架构如下：

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240323174300284.png" alt="image-20240323174300284" style="zoom: 67%;" />

#### **Producer(生产者) 和 Consumer(消费者)**

- **Producer(生产者)** :生产消息的一方（邮件投递者）
- **Consumer(消费者)** :消费消息的一方（邮件收件人）

消息一般由 2 部分组成：**消息头**（或者说是标签 Label）和 **消息体**。消息体也可以称为 payLoad ,消息体是不透明的，而消息头则由一系列的可选属性组成，这些属性包括 routing-key（路由键）、priority（相对于其他消息的优先权）、delivery-mode（指出该消息可能需要持久性存储）等。生产者把消息交由 RabbitMQ 后，RabbitMQ 会根据消息头把消息发送给感兴趣的 Consumer(消费者)。

#### **Exchange(交换器)**

在 RabbitMQ 中，消息并不是直接被投递到 **Queue(消息队列)** 中的，中间还必须经过 **Exchange(交换器)** 这一层，**Exchange(交换器)** 会把我们的消息分配到对应的 **Queue(消息队列)** 中。

**Exchange(交换器)** 用来接收生产者发送的消息并将这些消息路由给服务器中的队列中，如果路由不到，或许会返回给 **Producer(生产者)** ，或许会被直接丢弃掉 。这里可以将 RabbitMQ 中的交换器看作一个简单的实体。

**RabbitMQ 的 Exchange(交换器) 有 4 种类型，不同的类型对应着不同的路由策略**：**direct(默认)**，**fanout**, **topic**, 和 **headers**，不同类型的 Exchange 转发消息的策略有所区别。这个会在介绍 **Exchange Types(交换器类型)** 的时候介绍到。

Exchange(交换器) 示意图如下：

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240323174611673.png" alt="image-20240323174611673" style="zoom: 67%;" />

RabbitMQ 主要有 4 种交换机:

1. **Fanout Exchange**（广播类型）：Fanout交换机**将消息广播到其绑定的所有队列**。当消息被发送到Fanout交换机时，它会将消息复制到所有绑定的队列上，**而不考虑路由键的值**。因此，无论消息的路由键是什么，都会被广播到所有队列。Fanout交换机主要用于广播消息给所有的消费者。
2. **Direct Exchange**（直连类型）：Direct交换机是**根据消息的路由键选择将消息路由到与消息具有相同路由键绑定的队列**。例如，**当消息的路由键与绑定键完全匹配时**，消息将被路由到对应的队列。Direct交换机主要用于一对一的消息路由。
3. **Topic Exchange**（主题类型）：Topic交换机**将消息根据路由键的模式进行匹配，并将消息路由到与消息的路由键匹配的队列**。路由键可以**使用通配符匹配**，支持两种通配符符号，"#"表示匹配一个或多个单词，"*"表示匹配一个单词。Topic交换机主要用于灵活的消息路由。
4. **Headers Exchange**（头类型）：Headers交换机是**根据消息的头部信息进行匹配，并将消息路由到匹配的队列**。头部信息通常是**一组键值对**，可以使用各种自定义的标准和非标准的头部信息进行匹配。Headers交换机主要用于复杂的匹配规则。





#### **Queue(消息队列)**

**Queue(消息队列)** 用来保存消息直到发送给消费者。它是消息的容器，也是消息的终点。一个消息可投入一个或多个队列。消息一直在队列里面，等待消费者连接到这个队列将其取走。

**RabbitMQ** 中消息只能存储在 **队列** 中，这一点和 **Kafka** 这种消息中间件相反。Kafka 将消息存储在 **topic（主题）** 这个逻辑层面，而相对应的队列逻辑只是 topic 实际存储文件中的位移标识。 RabbitMQ 的生产者生产消息并最终投递到队列中，消费者可以从队列中获取消息并消费。

**多个消费者可以订阅同一个队列**，这时队列中的消息会被平均分摊（Round-Robin，即轮询）给多个消费者进行处理，而不是每个消费者都收到所有的消息并处理，这样避免消息被重复消费。

**RabbitMQ** 不支持队列层面的广播消费,如果有广播消费的需求，需要在其上进行二次开发,这样会很麻烦，不建议这样做。





#### **Broker（消息中间件的服务节点）**

对于 RabbitMQ 来说，一个 RabbitMQ Broker 可以简单地看作一个 RabbitMQ 服务节点，或者 RabbitMQ 服务实例。大多数情况下也可以将一个 RabbitMQ Broker 看作一台 RabbitMQ 服务器。

下图展示了生产者将消息存入 RabbitMQ Broker,以及消费者从 Broker 中消费数据的整个流程。

![消息队列的运转过程](https://oss.javaguide.cn/github/javaguide/rabbitmq/67952922.jpg)





### **什么是死信队列？**

DLX，全称为 `Dead-Letter-Exchange`，死信交换器，死信邮箱。当消息在一个队列中变成死信 (`dead message`) 之后，它能被重新被发送到另一个交换器中，这个交换器就是 DLX，绑定 DLX 的队列就称之为死信队列。

**导致的死信的几种原因**：

- 消息被拒（`Basic.Reject /Basic.Nack`) 且 `requeue = false`。
- 消息 TTL 过期。
- 队列满了，无法再添加。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240323190155110.png" alt="image-20240323190155110" style="zoom:50%;" />





### **什么是延迟队列？**

延迟队列指的是存储对应的延迟消息，消息被发送以后，并不想让消费者立刻拿到消息，而是等待特定时间后，消费者才能拿到这个消息进行消费。

RabbitMQ 本身是没有延迟队列的，要实现延迟消息，一般有两种方式：

1. 通过 RabbitMQ 本身队列的特性来实现，需要使用 RabbitMQ 的死信交换机（Exchange）和消息的存活时间 TTL（Time To Live）。
2. 在 RabbitMQ 3.5.7 及以上的版本提供了一个插件（rabbitmq-delayed-message-exchange）来实现延迟队列功能。同时，插件依赖 Erlang/OPT 18.0 及以上。

也就是说，**AMQP 协议以及 RabbitMQ 本身没有直接支持延迟队列的功能，但是可以通过 TTL 和 DLX 模拟出延迟队列的功能。**

延迟队列适用于许多场景，包括：

1. 定时任务：可以使用延迟队列来实现任务的定时触发，例如定时发送邮件或推送通知。
2. 消息重试：当某个消息失败后，可以将其放入延迟队列，并设置延迟时间，以便稍后重新投递。
3. 延迟通知：例如在某个时间后发送提醒通知。

延迟队列的优点有：

1. 灵活性：可以根据实际需求，灵活地设置延迟时间，适应各种业务场景。
2. 解耦性：延迟队列可以将消息发送与消费解耦，提高系统的可扩展性和稳定性。
3. 可靠性：通过延迟队列，可以确保消息在一定时间后被投递，降低消息丢失的风险。

延迟队列的缺点有：

1. 系统复杂性：引入延迟队列会增加系统的复杂性和维护成本。
2. 延迟时间不准确：由于网络延迟、系统负载等原因，延迟时间可能会有一定的误差。



### **RabbitMQ 消息怎么传输？**

由于 TCP 链接的创建和销毁开销较大，且并发数受系统资源限制，会造成性能瓶颈，所以 RabbitMQ 使用信道的方式来传输数据。信道（Channel）是生产者、消费者与 RabbitMQ 通信的渠道，信道是建立在 TCP 链接上的虚拟链接，且每条 TCP 链接上的信道数量没有限制。就是说 RabbitMQ 在一条 TCP 链接上建立成百上千个信道来达到多个线程处理，这个 TCP 被多个线程共享，每个信道在 RabbitMQ 都有唯一的 ID，保证了信道私有性，每个信道对应一个线程使用。



### **如何保证消息的可靠性？**

消息到 MQ 的过程中搞丢，MQ 自己搞丢，MQ 到消费过程中搞丢。

![image-20240323201143766](https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240323201143766.png)

#### **保证生产者发送消息到 RabbitMQ Server**

生产者到 RabbitMQ：事务机制和 Confirm 机制，注意：事务机制和 Confirm 机制是互斥的，两者不能共存，会导致 RabbitMQ 报错。

##### **事务机制**

配置类中配置事务管理器

```java
/**
 * 消息队列配置类
 *
 * @author 单程车票
 */
@Configuration
public class RabbitMQConfig {
    /**
     * 配置事务管理器
     */
    @Bean
    public RabbitTransactionManager transactionManager(ConnectionFactory connectionFactory) {
        return new RabbitTransactionManager(connectionFactory);
    }
}
```

通过添加事务注解 ＋ 开启事务实现事务机制

```java
/**
 * 消息业务实现类
 *
 * @author 单程车票
 */
@Service
public class RabbitMQServiceImpl {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Transactional // 事务注解
    public void sendMessage() {
        // 开启事务
        rabbitTemplate.setChannelTransacted(true);
        // 发送消息
        rabbitTemplate.convertAndSend(RabbitMQConfig.Direct_Exchange, routingKey, message);
    }
}
```

通过上面的配置即可实现事务机制，执行流程为：在生产者发送消息之前，开启事务，而后发送消息，如果消息发送至 RabbitMQ Server 失败后，进行事务回滚，重新发送。如果 RabbitMQ Server 接收到消息，则提交事务。

可以发现事务机制其实是**同步操作**，存在阻塞生产者的情况直到 RabbitMQ Server 应答，这样其实会很大程度上**降低发送消息的性能**，所以**一般不会使用事务机制来保证生产者的消息可靠性**，而是使用发送方确认机制。



##### **发送方确认机制**

配置文件

```java
spring:
  rabbitmq:
    publisher-confirm-type: correlated  # 开启发送方确认机制
```

配置属性有三种分别为：

- `none`：表示禁用发送方确认机制
- `correlated`：表示开启发送方确认机制
- `simple`：表示开启发送方确认机制，并支持 `waitForConfirms()` 和 `waitForConfirmsOrDie()` 的调用。

这里一般使用 `correlated` 开启发送方确认机制即可，至于 `simple` 的 `waitForConfirms()` 方法调用是指**串行确认方法**，即生产者发送消息后，调用该方法等待 RabbitMQ Server 确认，如果返回 false 或超时未返回则进行消息重传。由于串行性能较差，这里一般都是用异步 confirm 模式。


通过调用 `setConfirmCallback()` 实现异步 confirm 模式感知消息发送结果

```java
/**
 * 消息业务实现类
 *
 * @author 单程车票
 */
@Service
public class RabbitMQServiceImpl {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Override
    public void sendMessage() {
        // 发送消息
        rabbitTemplate.convertAndSend(RabbitMQConfig.Direct_Exchange, routingKey, message);
        // 设置消息确认回调方法
        rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            /**
             * MQ确认回调方法
             * @param correlationData 消息的唯一标识
             * @param ack 消息是否成功收到
             * @param cause 失败原因
             */
            @Override
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                // 记录日志
                log.info("ConfirmCallback...correlationData["+correlationData+"]==>ack:["+ack+"]==>cause:["+cause+"]");
                if (!ack) {
                    // 出错处理
                    ...
                }
            }
        });
    }
}
```

生产者发送消息后通过调用 `setConfirmCallback()` 可以将信道设置为 confirm 模式，所有消息会被指派一个消息唯一标识，当消息被发送到 RabbitMQ Server 后，Server 确认消息后生产者会回调设置的方法，从而实现生产者可以感知到消息是否正确无误的投递，从而实现发送方确认机制。并且该模式是异步的，发送消息的吞吐量会得到很大提升。

上面就是发送放确认机制的配置和使用，使用这种机制可以保证生产者的消息可靠性投递，并且性能较好。



#### **保证消息能从交换机路由到指定队列**

在确保生产者能将消息投递到交换机的前提下，RabbitMQ 同样提供了消息投递失败的策略配置来确保消息的可靠性，接下来通过配置来介绍一下消息投递失败的策略。

先说配置：

```text
spring:
  rabbitmq:
    publisher-confirm-type: correlated  # 开启发送方确认机制
    publisher-returns: true   # 开启消息返回
    template:
      mandatory: true     # 消息投递失败返回客户端
```

`mandatory` 分为 **true 失败后返回客户端** 和 **false 失败后自动删除**两种策略。显然设置为 false 无法保证消息的可靠性。

到这里的配置是可以保证生产者发送消息的可靠性投递。

通过调用 `setReturnCallback()` 方法设置路由失败后的回调方法：

```java
/**
 * 消息业务实现类
 *
 * @author 单程车票
 */
@Service
public class RabbitMQServiceImpl {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Override
    public void sendMessage() {
        // 发送消息
        rabbitTemplate.convertAndSend(RabbitMQConfig.Direct_Exchange, routingKey, message);
        // 设置消息确认回调方法
        rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            /**
             * MQ确认回调方法
             * @param correlationData 消息的唯一标识
             * @param ack 消息是否成功收到
             * @param cause 失败原因
             */
            @Override
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                // 记录日志
                log.info("ConfirmCallback...correlationData["+correlationData+"]==>ack:["+ack+"]==>cause:["+cause+"]");
                if (!ack) {
                    // 出错处理
                    ...
                }
            }
        });
        
        // 设置路由失败回调方法
        rabbitTemplate.setReturnCallback(new RabbitTemplate.ReturnCallback() {
            /**
             * MQ没有将消息投递给指定的队列回调方法
             * @param message 投递失败的消息详细信息
             * @param replyCode 回复的状态码
             * @param replyText 回复的文本内容
             * @param exchange 消息发给哪个交换机
             * @param routingKey 消息用哪个路邮键
             */
            @Override
            public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
                // 记录日志
                log.info("Fail Message["+message+"]==>replyCode["+replyCode+"]" +"==>replyText["+replyText+"]==>exchange["+exchange+"]==>routingKey["+routingKey+"]");
                // 出错处理
                ...
            }
        });
    }
}
```

通过调用 `setReturnCallback()` 方法即可实现当交换机路由到指定队列失败后回调方法，拿到被退回的消息信息，进行相应的处理如记录日志或重传等等。

#### **保证消息在 RabbitMQ Server 中的持久化**

对于消息的持久化，只需要在发送消息时将消息持久化，并且在创建交换机和队列时也保证持久化即可。

配置如下：

```java
/**
 * 消息队列
 */
@Bean
public Queue queue() {
    // 四个参数：name（队列名）、durable（持久化）、 exclusive（独占）、autoDelete（自动删除）
    return new Queue(MESSAGE_QUEUE, true);
}

/**
 * 直接交换机
 */
@Bean
public DirectExchange exchange() {
    // 四个参数：name（交换机名）、durable（持久化）、autoDelete（自动删除）、arguments（额外参数）
    return new DirectExchange(Direct_Exchange, true, false);
}
```

在创建交换机和队列时通过构造方法将持久化的参数都设置为 true 即可实现交换机和队列的持久化。

```java
@Override
public void sendMessage() {
    // 构造消息（将消息持久化）
    Message message = MessageBuilder.withBody("单程车票".getBytes(StandardCharsets.UTF_8)).setDeliveryMode(MessageDeliveryMode.PERSISTENT).build();
    // 向MQ发送消息（消息内容都为消息表记录的id）
    rabbitTemplate.convertAndSend(RabbitMQConfig.Direct_Exchange, routingKey, message);
}
```

在发送消息前通过调用 `MessageBuilder` 的 `setDeliveryMode(MessageDeliveryMode.PERSISTENT)` 在构造消息时设置消息持久化（`MessageDeliveryMode.PERSISTENT`）即可实现对消息的持久化。

通过确保消息、交换机、队列的持久化操作可以保证消息的在 RabbitMQ Server 中不丢失，从而保证可靠性，其实除了持久化之外还需要保证 RabbitMQ 的高可用性，否则 MQ 都宕机或磁盘受损都无法确保消息的可靠性





#### **保证消费者消费的消息不丢失**

##### **basicAck机制**

rabbitMQ提供手动确认方法，将消息队列设置为手动确认

```java
channel.basicConsume(TASK_QUEUE_NAME, false, deliverCallback, consumerTag -> {});
```

basicConsume()方法源码：

```
String basicConsume(String queue, boolean autoAck, DeliverCallback deliverCallback, CancelCallback cancelCallback) throws IOException;
```

`autoAck`：一个布尔值，表示是否自动确认消息。如果设置为 `true`，表示消费者在接收到消息后会自动确认消息，即告诉 RabbitMQ 服务器该消息已经被成功处理；如果设置为 `false`，表示消费者需要手动确认消息。在手动确认模式下，消费者需要在成功处理消息后发送确认给 RabbitMQ，否则消息不会被标记为已传递，仍然会保留在队列中。

同时可以指定拒绝某些消息

**指定确认某条消息**：

```java
channel.basicAck(delivery.getEnvelope().getDeliveryTag(),false);
```

源码

```
void basicAck(long deliveryTag, boolean multiple) throws IOException;
```

1. `deliveryTag`：表示要确认的消息的标识符。每个消息都有一个唯一的`deliveryTag`，用于标识消息的顺序。
2. `multiple`：表示是否批量确认消息。如果设置为`true`，则表示确认所有在`deliveryTag`之前的未确认消息；如果设置为`false`，则只确认当前`deliveryTag`的消息。

第二参数 multiple 批量确认：是指是否要一次性确认所有的历史消息直到当前这条

**指定拒绝某条消息**：

```java
channel.basicNack(delivery.getEnvelope().getDeliveryTag(),false,false);
```

源码

```
void basicNack(long deliveryTag, boolean multiple, boolean requeue)
            throws IOException;
```

1. `deliveryTag`：表示要否定确认的消息的标识符。每个消息都有一个唯一的`deliveryTag`，用于标识消息的顺序。
2. `multiple`：表示是否批量否定确认消息。如果设置为`true`，则表示否定所有在`deliveryTag`之前的未确认消息；如果设置为`false`，则只否定当前`deliveryTag`的消息。
3. `requeue`：表示是否将消息重新放回队列。如果设置为`true`，则消息将被重新放回队列并可以被其他消费者重新消费；如果设置为`false`，则消息将会被丢弃。

第三个参数表示是否重新入队，可用于重试



##### **消息重试机制**

除了消费者应答机制外，Spring Boot也提供了一种重试机制，只需要通过配置即可实现消息重试从而确保消息的可靠性，这里简单介绍一下：

```text
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: auto  # 开启自动确认消费机制
        retry:
          enabled: true # 开启消费者失败重试
          initial-interval: 5000ms # 初始失败等待时长为5秒
          multiplier: 1  # 失败的等待时长倍数（下次等待时长 = multiplier * 上次等待时间）
          max-attempts: 3 # 最大重试次数
          stateless: true # true无状态；false有状态（如果业务中包含事务，这里改为false）
```

通过配置在消费者的方法上如果执行失败或执行异常**只需要抛出异常（一定要出现异常才会触发重试，注意：不要捕获异常）** 即可实现消息重试，这样也可以保证消息的可靠性。



##### **死信队列**

通过配置死信队列，RabbitMQ 可以将无法被成功处理的消息转移到死信队列中，而不会丢失这些消息。然后，可以通过专门的消费者来处理死信队列中的消息，以进一步处理或记录这些消息，从而确保了消息的可靠性和安全性。



### **如何保证 RabbitMQ 高可用的？**

RabbitMQ 是比较有代表性的，因为是基于主从（非分布式）做高可用性的，我们就以 RabbitMQ 为例子讲解第一种 MQ 的高可用性怎么实现。RabbitMQ 有三种模式：单机模式、普通集群模式、镜像集群模式。

**单机模式**

Demo 级别的，一般就是你本地启动了玩玩儿的?，没人生产用单机模式。

**普通集群模式**

意思就是在多台机器上启动多个 RabbitMQ 实例，每个机器启动一个。你创建的 queue，只会放在一个 RabbitMQ 实例上，但是每个实例都同步 queue 的元数据（元数据可以认为是 queue 的一些配置信息，通过元数据，可以找到 queue 所在实例）。

你消费的时候，实际上如果连接到了另外一个实例，那么那个实例会从 queue 所在实例上拉取数据过来。这方案主要是提高吞吐量的，就是说让集群中多个节点来服务某个 queue 的读写操作。

**镜像集群模式**

这种模式，才是所谓的 RabbitMQ 的高可用模式。跟普通集群模式不一样的是，在镜像集群模式下，你创建的 queue，无论元数据还是 queue 里的消息都会存在于多个实例上，就是说，每个 RabbitMQ 节点都有这个 queue 的一个完整镜像，包含 queue 的全部数据的意思。**然后每次你写消息到 queue 的时候，都会自动把消息同步到多个实例的 queue 上。**RabbitMQ 有很好的管理控制台，就是在后台新增一个策略，这个策略是镜像集群模式的策略，指定的时候是可以要求数据同步到所有节点的，也可以要求同步到指定数量的节点，再次创建 queue 的时候，应用这个策略，就会自动将数据同步到其他的节点上去了。

这样的好处在于，你任何一个机器宕机了，没事儿，其它机器（节点）还包含了这个 queue 的完整数据，别的 consumer 都可以到其它节点上去消费数据。坏处在于，第一，这个性能开销也太大了吧，消息需要同步到所有机器上，导致网络带宽压力和消耗很重！RabbitMQ 一个 queue 的数据都是放在一个节点里的，镜像集群下，也是每个节点都放这个 queue 的完整数据。

















