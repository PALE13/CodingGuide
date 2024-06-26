## **Spring事务**



### **Spring 管理事务的方式有几种？**

- **编程式事务**：在代码中硬编码(在分布式系统中推荐使用) : 通过 `TransactionTemplate`或者 `TransactionManager` 手动管理事务，事务范围过大会出现事务未提交导致超时，**因此事务要比锁的粒度更小。**
- **声明式事务**：在 XML 配置文件中配置或者直接基于注解（单体应用或者简单业务系统推荐使用） : 实际是通过 AOP 实现**（基于`@Transactional` 的全注解方式使用最多）**



### **如何使用`@Transactional`**

在要开启事务的方法上使用`@Transactional`注解即可

```java
@Transactional(rollbackFor = Exception.class)
public void save() {
  ......
}
```

我们知道 Exception 分为运行时异常 RuntimeException 和非运行时异常。在`@Transactional`注解中如果不配置`rollbackFor`属性,那么事务只会在遇到`RuntimeException`的时候才会回滚,加上`rollbackFor=Exception.class`,可以让事务在遇到非运行时异常时也回滚。

`@Transactional` 注解一般可以作用在`类`或者`方法`上。

- **作用于类**：当把`@Transactional` 注解放在类上时，表示所有该类的 public 方法都配置相同的事务属性信息。
- **作用于方法**：当类配置了`@Transactional`，方法也配置了`@Transactional`，方法的事务会覆盖类的事务配置信息。

*





### **@Transactional的实现原理？**

其实事务操作是AOP的一个核心体现，当一个方法添加@Transactional注解之后，spring会基于这个类生成一个代理对象，会将这个代理对象作为bean，当使用这个代理对象的方法的时候，如果有事务处理，那么会先把事务的自动提交给关系，然后去执行具体的业务逻辑，如果执行逻辑没有出现异常，那么代理逻辑就会直接提交，如果出现任何异常情况，那么直接进行回滚操作，当然用户可以控制对哪些异常进行回滚操作。





### **@Transaction声明式事务的缺点**

在方法上增加@Transaction声明式事务

如果在一个方法中存在较多耗时的操作，**很容易引发长事务的问题，长事务会带来锁的竞争**，影响性能，导致数据库连接池被消耗尽，影响程序的正常执行

如果方法存在嵌套调用，而被嵌套调用的方法声明了@Transcation事务，这时候就会出现事务的传播行为，容易引起事务混乱，程序运行结果出现异常

@Transaction声明式事务是将事务控制逻辑放在注解中，如果项目中的复杂度增加，事务的控制会变得更加复杂，导致代码的可读性和维护性下降

所以有些公司会推荐使用使用编程式事务，这样可以更加灵活的去控制事务的范围，减少事务的锁定时间，提高系统的性能





### **什么是事务传播？**

事务传播（Transaction Propagation）是指在调用一个带有事务性质的方法时，该方法如何处理已存在的事务以及是否开启新的事务。Spring 框架提供了多种事务传播行为，用于控制事务在方法之间的传播方式，以满足不同的业务需求。

假设有两个方法：`method1()` 和 `method2()`，它们都被声明为事务性方法，并且 `method1()` 调用了 `method2()`。在这种情况下，就会涉及到事务的传播。

```java
@Service
public class MyService {

    @Autowired
    private SomeRepository someRepository;

    @Transactional
    public void method1() {
        // 执行一些业务逻辑
        someRepository.doSomething();

        // 调用 method2()
        method2();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void method2() {
        // 执行另一些业务逻辑
        someRepository.doSomethingElse();
    }
}
```

在上述代码中，`method1()` 和 `method2()` 都被声明为事务性方法。当 `method1()` 调用 `method2()` 时，就会涉及到事务传播的问题。

如果 `method1()` 和 `method2()` 都采用默认的事务传播行为 `REQUIRED`，那么在 `method1()` 开始时会开启一个新的事务，然后调用 `method2()` 时，会将当前事务传播给 `method2()`。因为 `method2()` 使用的是 `REQUIRES_NEW` 传播行为，所以会挂起当前事务，并为 `method2()` 开启一个新的事务，执行完毕后再恢复 `method1()` 的事务。

如果在调用 `method2()` 时 `method1()` 没有事务，那么 `method2()` 将会以非事务方式执行，因为它使用的是 `REQUIRES_NEW` 传播行为，会开启一个新的事务。而如果 `method1()` 已经存在事务，那么 `method2()` 将会以新的事务方式执行，因为它会挂起当前事务并开启一个新的事务。





### **Spring 事务中哪几种事务传播行为?**

**事务传播行为是为了解决业务层方法之间互相调用的事务问题**。

当事务方法被另一个事务方法调用时，必须指定事务应该如何传播。例如：方法可能继续在现有事务中运行，也可能开启一个新事务，并在自己的事务中运行。

正确的事务传播行为可能的值如下:

**`TransactionDefinition.PROPAGATION_REQUIRED`**

使用的最多的一个事务传播行为，我们平时经常使用的`@Transactional`注解默认使用就是这个事务传播行为。**如果外部存在事务，则加入该事务；如果外部没有事务，则创建一个新的事务。**常用于增删改

第一种情况

```java
@Transactional(propagation = Propagation.REQUIRED)
public void testMain(){
    A(a1);  //调用A入参a1
    testB();    //调用testB
}
@Transactional(propagation = Propagation.REQUIRED)
public void testB(){
    B(b1);  //调用B入参b1
    throw Exception;     //发生异常抛出
    B(b2);  //调用B入参b2
}
```

数据库没有插入新的数据，数据库还是保持着执行testMain方法之前的状态，没有发生改变。testMain上声明了事务，在执行testB方法时就加入了testMain的事务（**外部存在事务，则加入这个事务**），在执行testB方法抛出异常后事务会发生回滚，又testMain和testB使用的同一个事务，所以事务回滚后testMain和testB中的操作都会回滚，也就使得数据库仍然保持初始状态

第二种情况

```java
public void testMain(){
    A(a1);  //调用A入参a1
    testB();    //调用testB
}
@Transactional(propagation = Propagation.REQUIRED)
public void testB(){
    B(b1);  //调用B入参b1
    throw Exception;     //发生异常抛出
    B(b2);  //调用B入参b2
}
```

数据a1存储成功，数据b1和b2没有存储。由于testMain没有声明事务，testB有声明事务且传播行为是REQUIRED，所以在执行testB时会自己新建一个事务（**如果当前没有事务，则自己新建一个事务**），testB抛出异常则只有testB中的操作发生了回滚，也就是b1的存储会发生回滚，但a1数据不会回滚，所以最终a1数据存储成功，b1和b2数据没有存储



**`TransactionDefinition.PROPAGATION_SUPPORTS`**

 **如果外部存在事务，则加入该事务；如果外部没有事务，则以非事务的方式继续运行。**常用于查询

```java
public void testMain(){
    A(a1);  //调用A入参a1
    testB();    //调用testB
}
@Transactional(propagation = Propagation.SUPPORTS)
public void testB(){
    B(b1);  //调用B入参b1
    throw Exception;     //发生异常抛出
    B(b2);  //调用B入参b2
}
```

这种情况下，执行testMain的最终结果就是，a1，b1存入数据库，b2没有存入数据库。由于testMain没有声明事务，且testB的事务传播行为是SUPPORTS，所以执行testB时就是没有事务的（**如果当前没有事务，就以非事务方法执行**），则在testB抛出异常时也不会发生回滚，所以最终结果就是a1和b1存储成功，b2没有存储。



**`TransactionDefinition.PROPAGATION_REQUIRES_NEW`**

**不管外部存不存在事务，都创建事务**，常用内部和外部事务不关联的业务场景，如日志

`Propagation.REQUIRES_NEW`修饰的内部方法会新开启自己的事务，且开启的事务相互独立，互不干扰。

```java
@Transactional(propagation = Propagation.REQUIRED)
public void testMain(){
    A(a1);  //调用A入参a1
    testB();    //调用testB
    throw Exception;     //发生异常抛出
}
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void testB(){
    B(b1);  //调用B入参b1
    B(b2);  //调用B入参b2
}
```

这种情形的执行结果就是a1没有存储，而b1和b2存储成功，因为testB的事务传播设置为REQUIRES_NEW，所以在执行testB时会开启一个新的事务，testMain中发生的异常时在testMain所开启的事务中，所以这个异常不会影响testB的事务提交，testMain中的事务会发生回滚，所以最终a1就没有存储，而b1和b2就存储成功了。



**`TransactionDefinition.PROPAGATION_NESTED`**

如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；嵌套事务是一个单独的事务，它有自己的保存点，可以回滚到嵌套事务开始之前的状态。如果当前没有事务，则该取值等价于`TransactionDefinition.PROPAGATION_REQUIRED`。

第一种情况

```java
@Transactional(propagation = Propagation.REQUIRED)
public void testMain(){
    A(a1);  //调用A入参a1
    try{
        testB();    //调用testB
    }catch（Exception e){

    }
    A(a2);
}
@Transactional(propagation = Propagation.NESTED)
public void testB(){
    B(b1);  //调用B入参b1
    throw Exception;     //发生异常抛出
    B(b2);  //调用B入参b2
}
```

这种场景下，结果是a1,a2存储成功，b1和b2存储失败，因为调用方catch了被调方的异常，所以只有子事务回滚了。

第二种情况

```java
@Transactional(propagation = Propagation.REQUIRED)
public void testMain(){
    A(a1);  //调用A入参a1
    testB();    //调用testB
    throw Exception;     //发生异常抛出
}
@Transactional(propagation = Propagation.NESTED)
public void testB(){
    B(b1);  //调用B入参b1
    B(b2);  //调用B入参b2
}
```

该场景下，所有数据都不会存入数据库，因为在testMain发生异常时，父事务回滚则子事务也跟着回滚了



**`TransactionDefinition.PROPAGATION_MANDATORY`**如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。（mandatory：强制性），这个使用的很少

**`TransactionDefinition.PROPAGATION_NOT_SUPPORTED`**: 以非事务方式运行，如果当前存在事务，则把当前事务挂起。

**`TransactionDefinition.PROPAGATION_NEVER`**: 以非事务方式运行，如果当前存在事务，则抛出异常。





若是错误的配置以下 3 种事务传播行为，事务将不会发生回滚：

- **`TransactionDefinition.PROPAGATION_SUPPORTS`**: 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
- **`TransactionDefinition.PROPAGATION_NOT_SUPPORTED`**: 以非事务方式运行，如果当前存在事务，则把当前事务挂起。
- **`TransactionDefinition.PROPAGATION_NEVER`**: 以非事务方式运行，如果当前存在事务，则抛出异常。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240409204628518.png" alt="image-20240409204628518" style="zoom:50%;" />



### **Spring 事务中的隔离级别有哪几种?**

和事务传播行为这块一样，为了方便使用，Spring 也相应地定义了一个枚举类：`Isolation`

```java
public enum Isolation {

    DEFAULT(TransactionDefinition.ISOLATION_DEFAULT),
    READ_UNCOMMITTED(TransactionDefinition.ISOLATION_READ_UNCOMMITTED),
    READ_COMMITTED(TransactionDefinition.ISOLATION_READ_COMMITTED),
    REPEATABLE_READ(TransactionDefinition.ISOLATION_REPEATABLE_READ),
    SERIALIZABLE(TransactionDefinition.ISOLATION_SERIALIZABLE);

    private final int value;

    Isolation(int value) {
        this.value = value;
    }

    public int value() {
        return this.value;
    }

}
```

下面我依次对每一种事务隔离级别进行介绍：

- **`TransactionDefinition.ISOLATION_DEFAULT（默认）`** :使用后端数据库默认的隔离级别，MySQL 默认采用的 `REPEATABLE_READ` 隔离级别 Oracle 默认采用的 `READ_COMMITTED` 隔离级别.
- **`TransactionDefinition.ISOLATION_READ_UNCOMMITTED（读未提交）`** :最低的隔离级别，使用这个隔离级别很少，因为它允许读取尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**
- **`TransactionDefinition.ISOLATION_READ_COMMITTED（读已提交）`** : 允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**
- **`TransactionDefinition.ISOLATION_REPEATABLE_READ（可重复读）`** : 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生。**
- **`TransactionDefinition.ISOLATION_SERIALIZABLE（串行化）`** : 最高的隔离级别，完全服从 ACID 的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。但是这将严重影响程序的性能。通常情况下也不会用到该级别。



### **@Transactional(rollbackFor = Exception.class)注解了解吗？**

`Exception` 分为运行时异常 `RuntimeException` 和非运行时异常。事务管理对于企业应用来说是至关重要的，即使出现异常情况，它也可以保证数据的一致性。

当 `@Transactional` 注解作用于类上时，该类的所有 public 方法将都具有该类型的事务属性，同时，我们也可以在方法级别使用该标注来覆盖类级别的定义。

**`@Transactional` 注解默认回滚策略是只有在遇到`RuntimeException`(运行时异常) 或者 `Error` 时才会回滚事务，而不会回滚 `Checked Exception`（受检查异常）。**这是因为 Spring 认为`RuntimeException`和 Error 是不可预期的错误，而受检异常是可预期的错误，可以通过业务逻辑来处理。

如果想要修改默认的回滚策略，可以使用 `@Transactional` 注解的 `rollbackFor` 和 `noRollbackFor` 属性来指定哪些异常需要回滚，哪些异常不需要回滚。例如，如果想要让所有的异常都回滚事务，可以使用如下的注解：

```java
@Transactional(rollbackFor = Exception.class)
public void someMethod() {
// some business logic
}
```

如果想要让某些特定的异常不回滚事务，可以使用如下的注解：

```java
@Transactional(noRollbackFor = CustomException.class)
public void someMethod() {
// some business logic
}
```









