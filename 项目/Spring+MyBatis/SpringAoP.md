## SpirngAoP



### **谈谈自己对于 AOP 的了解？**

AOP(Aspect-Oriented Programming:面向切面编程)能够将那些**与业务无关，却为业务模块所共同调用的逻辑或责任**（例如事务处理、日志管理、权限控制等）封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可拓展性和可维护性。所谓切面，相当于应用对象间的横切点，我们可以将其单独抽象为单独的模块。

Spring AOP 就是**基于动态代理**的，如果要代理的对象，实现了某个接口，那么 Spring AOP 会使用 **JDK Proxy**，去创建代理对象，而对于没有实现接口的对象，就无法使用 JDK Proxy 去进行代理了，这时候 Spring AOP 会使用 **Cglib** 生成一个被代理对象的子类来作为代理，如下图所示：

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240312123012860.png" alt="image-20240312123012860" style="zoom: 67%;" />

当然你也可以使用 **AspectJ** ，Spring AOP 已经集成了 AspectJ ，AspectJ 应该算的上是 Java 生态系统中最完整的 AOP 框架了。

**JDK Proxy 和 Cglib的区别**

JDK Proxy：**JDK Proxy是基于Java的反射机制实现**的。它要求被代理的类实现一个或多个接口，然后通过java.lang.reflect.Proxy类和java.lang.reflect.InvocationHandler接口来创建代理对象和处理代理方法调用。JDK Proxy适用于代理接口的情况。它要求被代理的类实现接口，然后基于接口生成代理对象。由于Java只支持单继承，因此JDK Proxy无法直接代理没有实现接口的类。

Cglib：Cglib是**通过继承被代理类并生成子类**来实现的。它通过**字节码操作库（ASM或Byte Buddy）在运行时动态生成代理类的子类（运行时直接在内存中生成并加载修改后的字节码文件）**，并重写需要代理的方法。Cglib适用于代理类的情况。它可以直接代理没有实现接口的类，通过生成被代理类的子类来实现代理。


AOP 切面编程涉及到的一些专业术语：

| 术语              |                             含义                             |
| :---------------- | :----------------------------------------------------------: |
| 目标(Target)      |                         被通知的对象                         |
| 代理(Proxy)       |             向目标对象应用通知之后创建的代理对象             |
| 连接点(JoinPoint) |         目标对象的所属类中，定义的所有方法均为连接点         |
| 切入点(Pointcut)  | 被切面拦截 / 增强的连接点（切入点一定是连接点，连接点不一定是切入点） |
| 通知(Advice)      | 增强的逻辑 / 代码，也即拦截到目标对象的连接点之后要做的事情  |
| 切面(Aspect)      |                切入点(Pointcut)+通知(Advice)                 |
| Weaving(织入)     | 将通知应用到目标对象，进而生成代理对象的过程动作，分为编译器织入和运行时织入 |





### AOP 解决了什么问题？

OOP 不能很好地处理一些分散在多个类或对象中的公共行为（如日志记录、事务管理、权限控制、接口限流、接口幂等等），这些行为通常被称为 **横切关注点（cross-cutting concerns）** 。如果我们在每个类或对象中都重复实现这些行为，那么会导致代码的冗余、复杂和难以维护。

AOP 可以将横切关注点（如日志记录、事务管理、权限控制、接口限流、接口幂等等）从 **核心业务逻辑（core concerns，核心关注点）** 中分离出来，实现关注点的分离。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240420171113311.png" alt="image-20240420171113311" style="zoom:50%;" />







### **具体实现**

```java
// 定义一个接口
public interface UserService {
    void addUser(String username, String password);
    void deleteUser(String username);
}

// 实现接口的目标类
public class UserServiceImpl implements UserService {
    @Override
    public void addUser(String username, String password) {
        System.out.println("Adding user: " + username);
    }

    @Override
    public void deleteUser(String username) {
        System.out.println("Deleting user: " + username);
    }
}

// 定义一个切面类
@Aspect
@Component
public class LoggingAspect {
    //通知（切入点）
    @Before("execution(* com.example.UserService.*(..))")
    public void beforeAdvice(JoinPoint joinPoint) {
        System.out.println("Before method: " + joinPoint.getSignature().getName());
    }
}

// 配置文件中启用Spring AOP
@Configuration
@EnableAspectJAutoProxy
@ComponentScan(basePackages = "com.example")
public class AppConfig {
    // 这里可以添加其他配置
}

// 测试类
public class Main {
    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        UserService userService = context.getBean(UserService.class);
        userService.addUser("john", "password123"); // 这里会触发切面的beforeAdvice方法
        userService.deleteUser("john"); // 这里也会触发切面的beforeAdvice方法
    }
}
```



### **Spring AOP 和 AspectJ AOP 有什么区别？**

**Spring AOP 属于运行时增强，而 AspectJ 是编译时增强。** Spring AOP 基于代理(Proxying)，而 AspectJ **基于字节码操作(Bytecode Manipulation)。**

Spring AOP 已经集成了 AspectJ ，AspectJ 应该算的上是 Java 生态系统中最完整的 AOP 框架了。AspectJ 相比于 Spring AOP 功能更加强大，但是 Spring AOP 相对来说更简单，

如果我们的切面比较少，那么两者性能差异不大。但是，当切面太多的话，最好选择 AspectJ ，它比 Spring AOP 快很多。





### **AspectJ 定义的通知类型有哪些？**

- **Before**（前置通知）：目标对象的方法调用之前触发
- **After** （后置通知）：目标对象的方法调用之后触发
- **AfterReturning**（返回通知）：目标对象的方法调用完成，在返回结果值之后触发
- **AfterThrowing**（异常通知）：目标对象的方法运行中抛出 / 触发异常后触发。AfterReturning 和 AfterThrowing 两者互斥。如果方法调用成功无异常，则会有返回值；如果方法抛出了异常，则不会有返回值。
- **Around** （环绕通知）：编程式控制目标对象的方法调用。环绕通知是所有通知类型中可操作范围最大的一种，因为它可以直接拿到目标对象，以及要执行的方法，所以环绕通知可以任意的在目标对象的方法调用前后搞事，甚至不调用目标对象的方法



### **如何实现环绕通知？**

在Spring AOP中，环绕通知（Around Advice）是一种比较灵活的通知类型，它可以在目标方法执行前后执行额外的逻辑，并且还可以决定是否调用目标方法。
在Spring AOP中，环绕通知（Around Advice）是一种比较灵活的通知类型，它可以在目标方法执行前后执行额外的逻辑，并且还可以决定是否调用目标方法。下面是使用环绕通知的步骤：

1. **定义环绕通知方法**：首先，需要在切面类中定义一个方法，该方法使用 `@Around` 注解标记，以表示它是一个环绕通知。方法的签名必须是 `public Object` 类型，并且接受一个 `ProceedingJoinPoint` 类型的参数。在方法内部可以编写环绕通知的逻辑，并通过 `ProceedingJoinPoint` 对象来控制目标方法的执行。
2. **编写环绕通知逻辑**：在环绕通知方法内部，可以编写额外的逻辑，例如日志记录、性能监控、事务管理等。可以在目标方法执行前执行预处理逻辑，在目标方法执行后执行后续处理逻辑，并且可以决定是否调用 `ProceedingJoinPoint` 对象的 `proceed()` 方法来执行目标方法。
3. **指定切入点表达式**：在 `@Around` 注解中指定切入点表达式，以确定环绕通知应该织入到哪些目标方法中。

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class PerformanceAspect {

    @Around("execution(* com.example.MyService.*(..))")
    public Object measureExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long startTime = System.nanoTime();
        Object result = joinPoint.proceed(); // 调用目标方法
        long endTime = System.nanoTime();
        long executionTime = endTime - startTime;

        System.out.println("Method " + joinPoint.getSignature().getName() + " execution time: " + executionTime + " nanoseconds");
        return result;
    }
}
```





### **多个切面的执行顺序如何控制？**

1、通常使用`@Order` 注解直接定义切面顺序

```java
// 值越小优先级越高
@Order(3)
@Component
@Aspect
public class LoggingAspect implements Ordered {
```

2、实现`Ordered` 接口重写 `getOrder` 方法。

```java
@Component
@Aspect
public class LoggingAspect implements Ordered {

    // ....

    @Override
    public int getOrder() {
        // 返回值越小优先级越高
        return 1;
    }
}
```









