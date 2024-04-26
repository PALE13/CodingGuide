## **SpringIoC**

### **谈谈自己对于 Spring IoC 的了解**

**IoC（Inversion of Control:控制反转）** 是一种设计思想，而不是一个具体的技术实现。IoC 的思想就是将原本在程序中手动创建对象的控制权，交由 Spring 框架来管理。不过， IoC 并非 Spring 特有，在其他语言中也有应用。

**为什么叫控制反转？**

- **控制**：指的是对象创建（实例化、管理）的权力
- **反转**：控制权交给外部环境（Spring 框架、IoC 容器）

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240319163827940.png" alt="image-20240319163827940" style="zoom: 67%;" />

将对象之间的相互依赖关系交给 IoC 容器来管理，并由 IoC 容器完成对象的注入。这样可以很大程度上简化应用的开发，把应用从复杂的依赖关系中解放出来。 IoC 容器就像是一个工厂一样，当我们需要创建一个对象的时候，只需要配置好配置文件/注解即可，完全不用考虑对象是如何被创建出来的。

在实际项目中一个 **Service 类可能依赖了很多其他的类，假如我们需要实例化这个 Service，你可能要每次都要搞清这个 Service 所有底层类的构造函数**，这可能会把人逼疯。如果利用 IoC 的话，你只需要配置好，然后在需要的地方引用就行了，这大大增加了项目的可维护性且降低了开发难度。

在 Spring 中， IoC 容器是 Spring 用来实现 IoC 的载体， IoC 容器实际上就是个 Map（key，value），Map 中存放的是各种对象。

Spring 时代我们一般通过 XML 文件来配置 Bean，后来开发人员觉得 XML 文件来配置不太好，于是 SpringBoot 注解配置就慢慢开始流行起来。





### **IOC的工作流程**

**第一个阶段：容器的初始化**

找到定义Bean的XML，注解，配置类

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240417115931072.png" alt="image-20240417115931072" style="zoom: 67%;" />

**通过解析和加载后生成BeanDefinition**，然后把BeanDefinition注册到IOC容器中

通过注解或者XML声明的Bean，都会解析的到一个BeanDefiniton实体，这个实体里面会包含bean的一些定义和基本的属性，最后把BeanDefinition保存到一个Map集合中，从而完成IOC容器的初始化 

**第二个阶段：完成Bean的初始化和依赖注入**

通过反射对没有设置lazy-init属性的单例bean进行初始化，然后完成依赖注入

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240417120046322.png" alt="image-20240417120046322" style="zoom: 67%;" />

最后就是Bean的使用

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240417120154665.png" alt="image-20240417120154665" style="zoom: 67%;" />

通常会通过@Autowired或者BeanFactory.getBean()从IOC容器中获取一个指定Bean的实例

针对设置了lazy-init的属性和非单例Bean的实例化，是在每一次获取bean对象的时候，调用bean的初始化方法来完成实例化的，并且SpringIOC容器不会去管理这些Bean





### **什么是 Spring Bean？**

简单来说，Bean 代指的就是那些**被 IoC 容器所管理的对象。**

我们需要告诉 IoC 容器帮助我们管理哪些对象，这个是通过配置元数据来定义的。配置元数据可以是 XML 文件、注解或者 Java 配置类。

```xml
<!-- Constructor-arg with 'value' attribute -->
<bean id="..." class="...">
   <constructor-arg value="..."/>
</bean>
```

下图简单地展示了 IoC 容器如何使用配置元数据来管理对象。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240312111716723.png" alt="image-20240312111716723" style="zoom:50%;" />



### **将一个类声明为 Bean 的注解有哪些?**

- `@Component`：通用的注解，可标注任意类为 `Spring` 组件。如果一个 Bean 不知道属于哪个层，可以使用`@Component` 注解标注。
- `@Repository` : 对应持久层即 Dao 层，主要用于数据库相关操作。
- `@Service` : 对应服务层，主要涉及一些复杂的逻辑，需要用到 Dao 层。
- `@Controller` : 对应 Spring MVC 控制层，主要用于接受用户请求并调用 `Service` 层返回数据给前端页面。





###  **@Component 和 @Bean 的区别是什么？**

- `@Component` 注解作用于类，而`@Bean`注解作用于方法。

**@Bean注解：**

- 使用@Bean注解可以在Java配置类（如@Configuration注解标记的类）中手动定义和配置Bean。
- @Bean注解适用于需要进行复杂配置、初始化、依赖注入等操作的情况。
- 通过@Bean注解手动创建的Bean可以在应用程序中进行灵活的定制和控制。

`@Bean`注解使用示例：

```java
@Configuration
public class AppConfig {
    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl();
    }
}
```

上面的代码相当于下面的 xml 配置

```xml
<beans>
    <bean id="transferService" class="com.acme.TransferServiceImpl"/>
</beans>
```

下面这个例子是通过 `@Component` 无法实现的。

```java
@Bean
public OneService getService(status) {
    case (status)  {
        when 1:
                return new serviceImpl1();
        when 2:
                return new serviceImpl2();
        when 3:
                return new serviceImpl3();
    }
}
```



**@Component注解：**

- 使用@Component注解可以将一个类标识为Spring的组件，**并由Spring容器自动进行扫描和管理。**
- @Component注解适用于简单的类组件，用于实现依赖注入和自动化的Bean管理。
- 通过@Component注解标识的类会被Spring自动扫描并创建对应的Bean，无需手动配置。

```
@Component
public class MyService {
    // Class body
}
```





### **@Autowired 和 @Resource 的区别是什么？**

**@Autowired** 

```java
public @interface Autowired {
    boolean required() default true;
}
```

`Autowired` 属于 Spring 内置的注解，**默认的注入方式为`byType`（根据类型进行匹配）**，也就是说会优先根据接口类型去匹配并注入 Bean （接口的实现类），**如果无法按照类型匹配，注入方式就会变为byName**

**这会有什么问题呢？** **当一个接口存在多个实现类的话，`byType`这种方式就无法正确注入对象了**，因为这个时候 Spring 会同时找到多个满足条件的选择，默认情况下它自己不知道选择哪一个。

**这种情况下，注入方式会变为 `byName`（根据名称进行匹配）**，这个名称通常就是类名（首字母小写）。就比如说下面代码中的 `smsService` 就是我这里所说的名称，这样应该比较好理解了吧。

```java
// smsService 就是我们上面所说的名称
@Autowired
private SmsService smsService;
```

举个例子，`SmsService` 接口有两个实现类: `SmsServiceImpl1`和 `SmsServiceImpl2`，且它们都已经被 Spring 容器所管理。

**也可以通过 `@Qualifier` 注解来显式指定名称而不是依赖变量的名称**

```java
// 报错，byName 和 byType 都无法匹配到 bean
@Autowired
private SmsService smsService;

// 正确注入 SmsServiceImpl1 对象对应的 bean
@Autowired
private SmsService smsServiceImpl1;

// 正确注入  SmsServiceImpl1 对象对应的 bean
// smsServiceImpl1 就是我们上面所说的名称
@Autowired
@Qualifier(value = "smsServiceImpl1")
private SmsService smsService;
```

我们还是建议通过 `@Qualifier` 注解来显式指定名称而不是依赖变量的名称。



 **@Resource**

`@Resource`属于 JDK 提供的注解，**默认注入方式为 `byName`。如果无法通过名称匹配到对应的 Bean 的话，注入方式会变为`byType`。**

`@Resource` 有两个比较重要且日常开发常用的属性：`name`（名称）、`type`（类型）。

```java
public @interface Resource {
    String name() default "";
    Class<?> type() default Object.class;
}
```

如果仅指定 `name` 属性则注入方式为`byName`，如果仅指定`type`属性则注入方式为`byType`，如果同时指定`name` 和`type`属性（不建议这么做）则注入方式为`byType`+`byName`。

```java
// 报错，byName 和 byType 都无法匹配到 bean
@Resource
private SmsService smsService;

// 正确注入 SmsServiceImpl1 对象对应的 bean
@Resource
private SmsService smsServiceImpl1;

// 正确注入 SmsServiceImpl1 对象对应的 bean（比较推荐这种方式）
@Resource(name = "smsServiceImpl1")
private SmsService smsService;
```

简单总结一下：

- `@Autowired` 是 Spring 提供的注解，`@Resource` 是 JDK 提供的注解。
- `Autowired` 默认的注入方式为`byType`（根据类型进行匹配），`@Resource`默认注入方式为 `byName`（根据名称进行匹配）。
- 当一个接口存在多个实现类的情况下，`@Autowired` 和`@Resource`都需要通过名称才能正确匹配到对应的 Bean。`Autowired` 可以通过 `@Qualifier` 注解来显式指定名称，`@Resource`可以通过 `name` 属性来显式指定名称。
- `@Autowired` 支持在构造函数、方法、字段和参数上使用。`@Resource` 主要用于字段和方法上的注入，不支持在构造函数或参数上使用。
- 在默认情况下，`@Autowired` 注解的属性是必须的，如果找不到匹配的bean，将会抛出异常。但是可以通过将 **`required` 属性设置为 `false` 来将其标记为可选的**。`@Resource` 注解的属性是可选的，如果找不到匹配的bean，则会尝试按名称查找。如果找不到匹配的bean，也会抛出异常，除非将其 `required` 属性设置为 `false`。







### **Bean 的作用域有哪些?**

Spring 中 Bean 的作用域通常有下面几种：

- **singleton** : IoC 容器中只有唯一的 bean 实例。Spring 中的 bean 默认都是单例的，是对单例设计模式的应用。
- **prototype** : 每次获取都会创建一个新的 bean 实例。也就是说，连续 `getBean()` 两次，得到的是不同的 Bean 实例。
- **request** （仅 Web 应用可用）: 每一次 HTTP 请求都会产生一个新的 bean（请求 bean），该 bean 仅在当前 HTTP request 内有效。
- **session** （仅 Web 应用可用） : 每一次来自新 session 的 HTTP 请求都会产生一个新的 bean（会话 bean），该 bean 仅在当前 HTTP session 内有效。
- **application/global-session** （仅 Web 应用可用）：每个 Web 应用在启动时创建一个 Bean（应用 Bean），该 bean 仅在当前应用启动时间内有效。
- **websocket** （仅 Web 应用可用）：每一次 WebSocket 会话产生一个新的 bean。

**如何配置 bean 的作用域呢？**

xml 方式：

```xml
<bean id="..." class="..." scope="singleton"></bean>
```

注解方式：

```java
@Bean
@Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public Person personPrototype() {
    return new Person();
}
```



### **Bean 是线程安全的吗？**

Spring 框架中的 Bean 是否线程安全，取决于其作用域和状态。

我们这里以最常用的两种作用域 prototype 和 singleton 为例介绍。几乎所有场景的 Bean 作用域都是使用默认的 singleton ，重点关注 singleton 作用域即可。

**prototype 作用域下，每次获取都会创建一个新的 bean 实例，不存在资源竞争问题**，所以不存在线程安全问题。singleton 作用域下，IoC 容器中只有唯一的 bean 实例，可能会存在资源竞争问题（取决于 Bean 是否有状态）。如果这个 bean 是有状态的话，那就存在线程安全问题（有状态 Bean 是指包含可变的成员变量的对象）。

**在singleton作用域下，如果定义了成员变量，会产生线程不安全问题**

不过，大部分 Bean 实际都是无状态（没有定义可变的成员变量）的（比如 Dao、Service），这种情况下， Bean 是线程安全的。

对于有状态单例 Bean 的线程安全问题

比如以下这个Bean，会对成员变量进行修改，就会产生不一致问题

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240417152009972.png" alt="image-20240417152009972" style="zoom: 67%;" />

常见的有三种解决办法：

1. 在 Bean 中尽量避免定义可变的成员变量。
2. 把成员变量声明到方法中
3. 在类中定义一个 `ThreadLocal` 成员变量，将需要的可变成员变量保存在 `ThreadLocal` 中（推荐的一种方式）。





### **Bean 的生命周期**

**创建 Bean 的实例**：Bean 容器首先会找到配置文件中的 Bean 定义，然后使用 Java 反射 API 来创建 Bean 的实例。

**Bean 属性赋值/填充**：为 Bean 设置相关属性和依赖，例如`@Autowired` 等注解注入的对象、`@Value` 注入的值、`setter`方法或构造函数注入依赖和值、`@Resource`注入的各种资源。

**Bean 初始化**： 

- 调用 Bean 实现的 Aware 接口的相关方法，Aware 接口是一组特定用途的接口，通过实现这些接口，Bean 可以获取到 Spring 容器的一些特定资源或者状态信息。
  - 如果 Bean 实现了 `BeanNameAware` 接口，调用 `setBeanName()`方法，传入 Bean 的名字。
  - 如果 Bean 实现了 `BeanClassLoaderAware` 接口，调用 `setBeanClassLoader()`方法，传入 `ClassLoader`对象的实例。
  - 如果 Bean 实现了 `BeanFactoryAware` 接口，调用 `setBeanFactory()`方法，传入 `BeanFactory`对象的实例。
  - 与上面的类似，如果实现了其他 `*.Aware`接口，就调用相应的方法。

- 如果有和加载这个 Bean 的 Spring 容器相关的 `BeanPostProcessor` 对象，执行`postProcessBeforeInitialization()` 方法
- 如果 Bean 实现了`InitializingBean`接口，执行`afterPropertiesSet()`方法。
- 如果 Bean 在配置文件中的定义包含 `init-method` 属性，执行指定的方法。
- 如果有和加载这个 Bean 的 Spring 容器相关的 `BeanPostProcessor` 对象，执行`postProcessAfterInitialization()` 方法。

**销毁 Bean**：销毁并不是说要立马把 Bean 给销毁掉，而是把 Bean 的销毁方法先记录下来，将来需要销毁 Bean 或者销毁容器的时候，就调用这些方法去释放 Bean 所持有的资源。 

- 如果 Bean 实现了 `DisposableBean` 接口，执行 `destroy()` 方法。
- 如果 Bean 在配置文件中的定义包含 `destroy-method` 属性，执行指定的 Bean 销毁方法。或者，也可以直接通过`@PreDestroy` 注解标记 Bean 销毁之前执行的方法。

![image-20240421185318570](https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240421185318570.png)

这个过程是由Spring容器自动管理的其中有两个环节我们可以进行干预。

我们可以自定义初始化方法，并在该方法前增加@PostConstruct注解。届时Spring容器将在调用SetBeanFactory方法之后调用该方法。

我们可以自定义销毁方法，并在该方法前增加@PreDestroy注解，届时Spring容器将在自身销毁前，调用这个方法。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240312121734960.png" alt="image-20240312121734960" style="zoom:50%;" />

**如何记忆呢？**

1. 整体上可以简单分为四步：实例化 —> 属性赋值 —> 初始化 —> 销毁。
2. 初始化这一步涉及到的步骤比较多，包含 `Aware` 接口的依赖注入、`BeanPostProcessor` 在初始化前后的处理以及 `InitializingBean` 和 `init-method` 的初始化操作。
3. 销毁这一步会注册相关销毁回调接口，最后通过`DisposableBean` 和 `destory-method` 进行销毁。





#### **源码分析**

`AbstractAutowireCapableBeanFactory` 的 `doCreateBean()` 方法中能看到依次执行了这 4 个阶段：

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
    throws BeanCreationException {

    // 1. 创建 Bean 的实例
    BeanWrapper instanceWrapper = null;
    if (instanceWrapper == null) {
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }

    Object exposedObject = bean;
    try {
        // 2. Bean 属性赋值/填充
        populateBean(beanName, mbd, instanceWrapper);
        // 3. Bean 初始化
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }

    // 4. 销毁 Bean-注册回调接口
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }

    return exposedObject;
}

```

`Aware` 接口能让 Bean 能拿到 Spring 容器资源。

Spring 中提供的 `Aware` 接口主要有：

1. `BeanNameAware`：注入当前 bean 对应 beanName；
2. `BeanClassLoaderAware`：注入加载当前 bean 的 ClassLoader；
3. `BeanFactoryAware`：注入当前 `BeanFactory` 容器的引用。



#### **InitializingBean**

Bean 实例化、属性注入完成后执行，InitializingBean 和 init-method 是 Spring 为 Bean 初始化提供的扩展点。

afterPropertiesSet它是InitializingBean的一个回调方法，可以在bean属性设置完成之后执行一些特定的操作。

afterPropertiesSet 方法通常用于执行那些不能在配置文件中完成的初始化操作，**需要借助外界才能完成初始化，如数据库连接池的相关初始化等**。因为在Spring容器初始化bean时，在bean属性设置完成后，会在执行其他初始化方法之前先执行afterPropertiesSet方法。

```java
public interface InitializingBean {
 // 初始化逻辑
	void afterPropertiesSet() throws Exception;
}
```

**可以注解 `@PostConstruct` 来代替 `afterPropertiesSet` 方法**



#### **BeanPostProcessor**

`BeanPostProcessor` 接口是 Spring 为修改 Bean 提供的强大扩展点。它**可以在Spring容器实例化bean之后，在执行bean的初始化方法前后，允许我们自定义修改新的bean实例，如修改bean的属性，可以给bean生成一个动态代理实例等等**，Spring AOP的底层处理也是通过实现BeanPostProcessor来执行代理包装逻辑的。

```java
public interface BeanPostProcessor {
	// 初始化前置处理
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	// 初始化后置处理
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
}
```

`postProcessBeforeInitialization`：`InitializingBean#afterPropertiesSet`方法以及自定义的 `init-method` 方法之前执行；

`postProcessAfterInitialization`：类似于上面，不过是在 `InitializingBean#afterPropertiesSet`方法以及自定义的 `init-method` 方法之后执行。



**使用方式**

Spring在初始化bean之后，就会遍历并执行所有的BeanPostProcessor中的postProcessAfterInitialization方法中的逻辑

```java
// 注意：必须将自己定义的MyBeanPostProcessor也加入到spring容器中去，这样Spring才能发现我们实现的这个post processor
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {

	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		// 可以针对某一个bean进行处理！否则默认会对所有bean生效
		if ("userService".equals(beanName)) {
			System.out.println("执行 userService 初始化之前的方法...");
		}
		return bean;
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if ("userService".equals(beanName)) {
			System.out.println("执行 userService 初始化之后的方法...");
		}
		return bean;
	}
}

```

当定义多个BeanPostProcessor的时候，如果想要自定义各个BeanPostProcessor的执行顺序，也可以实现Orderd接口，执行order属性的值，order越小，优先级越高，BeanPostProcessor最先被执行。





