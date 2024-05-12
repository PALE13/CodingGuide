## **SpringIOC**

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
- **prototype** : 每次获取都会创建一个新的 bean 实例。也就是说，连续 `getBean()` 两次，得到的是不同的 Bean 实例。原型Bean的生命周期由客户端管理，Spring容器不负责管理其生命周期。每次请求原型Bean时都会创建一个新的实例，Spring容器不会对其进行缓存或单例化处理。
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

**启动容器：**首先Spring容器启动之后，会根据使用不同类型的ApplicationContext，通过不同的方式去加载Bean配置，如xml方式、注解方式，将这些Bean配置加载到容器中，**作为Bean定义包装成BeanDefinition对象保存起来**，为下一步创建Bean做准备。

**创建 Bean 的实例**：根据加载的Bean定义信息，**通过反射来创建Bean实例**，如果是普通Bean，则直接创建Bean，如果是FactoryBean，说明真正要创建的对象为getObject()的返回值，调用getObject()将返回值作为Bean。

**Bean 属性赋值/填充**：为 Bean 设置相关属性和依赖，例如`@Autowired` 等注解注入的对象、`@Value` 注入的值、`setter`方法或构造函数注入依赖和值、`@Resource`注入的各种资源。

**Bean 初始化**： 

- 调用 Bean 实现的 Aware 接口的相关方法，Aware 接口是一组特定用途的接口，通过实现这些接口，Bean 可以获取到 Spring 容器的一些特定资源或者状态信息。
  - 如果 Bean 实现了 `BeanNameAware` 接口，调用 `setBeanName()`方法，传入 Bean 的名字。
  - 如果 Bean 实现了 `BeanClassLoaderAware` 接口，调用 `setBeanClassLoader()`方法，传入 `ClassLoader`对象的实例。
  - 如果 Bean 实现了 `BeanFactoryAware` 接口，调用 `setBeanFactory()`方法，传入 `BeanFactory`对象的实例。
  - 与上面的类似，如果实现了其他 `*.Aware`接口，就调用相应的方法。
- 如果有和加载这个 Bean 的 Spring 容器相关的 `BeanPostProcessor` 对象，执行`postProcessBeforeInitialization()` 方法
- 如果 Bean 实现了`InitializingBean`接口，执行`afterPropertiesSet()`方法。afterPropertiesSet 方法通常用于执行那些不能在配置文件中完成的初始化操作，**需要借助外界才能完成初始化，如数据库连接池的相关初始化等。**
- 如果 Bean 在配置文件中的定义包含 `init-method` 属性，执行指定的方法。
- 如果有和加载这个 Bean 的 Spring 容器相关的 `BeanPostProcessor` 对象，执行`postProcessAfterInitialization()` 方法。

**使用Bean：**

- 如果Bean为单例的话，那么容器会返回Bean给用户，并存入缓存池。
- 如果Bean是多例的话，容器将Bean返回给用户，剩下的生命周期由用户控制（手动销毁或通过JVM GC）。



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





### **BeanFactory和FactoryBean的区别？**

**BeanFactory**：是所有Spring Bean的容器根接口，给Spring 的容器定义一套规范，给IOC容器提供了一套完整的规范，比如我们常用到的getBean方法等

**FactoryBean**：是一个bean。但是他不是一个普通的bean，是可以创建对象的bean。。通常是用来创建比较复杂的bean，一般的bean 直接用xml配置即可，但如果一个bean的创建过程中涉及到很多其他的bean 和复杂的逻辑，直接用xml配置比较麻烦，这时可以考虑用FactoryBean，可以隐藏实例化复杂Bean的具体的细节.







### **BeanFactory和ApplicationContext有什么区别？**

BeanFactory和ApplicationContext是Spring的两大核心接口，都可以当做Spring的容器。其中ApplicationContext是BeanFactory的子接口。

两者区别如下：

1、**功能上的区别。**

BeanFactory是Spring里面最底层的接口，包含了各种Bean的定义，读取bean配置文档，管理bean的加载、实例化，控制bean的生命周期，维护bean之间的依赖关系。

ApplicationContext接口作为BeanFactory的派生，除了提供BeanFactory所具有的功能外，还提供了更完整的框架功能，如继承MessageSource、支持国际化、统一的资源文件访问方式、同时加载多个配置文件等功能。

2、**加载方式的区别。**

BeanFactroy采用的是延迟加载形式来注入Bean的，**即只有在使用到某个Bean时(调用getBean())，才对该Bean进行加载实例化。**这样，我们就不能发现一些存在的Spring的配置问题。如果Bean的某一个属性没有注入，BeanFacotry加载后，直至第一次使用调用getBean方法才会抛出异常。

**而ApplicationContext是在容器启动时，一次性创建了所有的Bean。**这样，在容器启动时，我们就可以发现Spring中存在的配置错误，这样有利于检查所依赖属性是否注入。 ApplicationContext启动后预载入所有的单例Bean，那么在需要的时候，不需要等待创建bean，因为它们已经创建好了。

相对于基本的BeanFactory，ApplicationContext 唯一的不足是占用内存空间。当应用程序配置Bean较多时，程序启动较慢。

ApplicationContext除了继承了BeanFactory外，还继承了ApplicationEventPublisher（事件发布器）、
ResouresPatternResolver（资源解析器）、MessageSource（消息资源）等。但是ApplicationContext的核心功能还是BeanFactory。

![image-20240511160427427](https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240511160427427.png)





### **怎么让2个bean按顺序加载？**

当一个 bean 需要在另一个 bean 初始化之后再初始化时，可以使用@DependOn注解





### **Spring的Bean是否会被JVM的GC回收**

Spring的单例（singleton）bean在Spring容器运行期间通常**不会被JVM的垃圾回收器（GC）回收**，而原型（prototype）bean则可能在不再被引用时被回收。具体如下：

1. **单例（Singleton）Beans**：默认情况下，Spring容器中创建的bean是单例的，即在整个应用程序生命周期内只存在一个实例。这些单例bean通常与Spring容器具有相同的生命周期，除非Spring容器被关闭或bean被显式销毁，否则它们会一直存在于内存中，因此不会被JVM的GC回收。
2. **原型（Prototype）Beans**：与单例bean不同，原型bean每次请求时都会创建一个新的实例。这意味着它们的生命周期通常较短，使用完毕后如果没有其他引用指向它们，就可能会被JVM的GC回收。
3. **作用域和生命周期**：Spring bean的作用域和生命周期也会影响其是否被回收。例如，如果一个bean被配置为在某个特定的生命周期结束或某个条件满足时销毁，那么它将可能被JVM回收。
4. **弱引用和软引用**：如果在Spring配置中使用了弱引用或软引用，那么即使bean仍然在Spring容器中，也可能在JVM的下一次GC周期中被回收，因为弱引用和软引用的对象更容易被GC处理。
5. **Spring容器关闭**：当Spring容器关闭时，所有的单例bean都会被销毁，此时它们将变得不可达，从而可能被JVM的GC回收。

综上所述，Spring的bean是否会被JVM的GC回收取决于其作用域、生命周期以及Spring容器的状态。单例bean在Spring容器运行期间通常不会被回收，而原型bean和其他特定条件下的bean可能会被回收



### **SpringBoot中不想加载一个bean如何做**

要在Spring Boot中阻止一个bean被加载，您可以采取以下几种方法：

**自定义@ComponentScan**：通过自定义@ComponentScan注解，您可以指定需要扫描的包路径，从而排除不想加载的bean。在@ComponentScan注解中使用excludeFilters属性来排除特定的类。

**使用@SpringBootApplication的exclude属性**：如果您使用的是@SpringBootApplication注解，可以利用它的exclude属性来排除自动配置类。例如，如果您不想加载某个自动配置类，可以在@SpringBootApplication注解中列出这个类。

**使用@EnableAutoConfiguration的exclude属性**：在Spring Boot的启动类上使用@EnableAutoConfiguration(exclude = {ClassNotToLoad.class})，这样可以排除掉不需要自动配置的类。

**使用TypeExcludeFilter**：创建一个自定义的TypeExcludeFilter实现，并在@ComponentScan注解中引用它。这样可以实现更精细的控制，只排除特定类型的bean。

**移除相关注解**：如果某个bean是通过@Component、@Service、@Repository等注解自动注册到Spring容器中的，您可以通过移除这些注解来阻止它的加载。

**调整bean加载顺序**：在某些情况下，您可能需要控制bean的加载顺序。虽然@Order、@AutoConfigureOrder、@AutoConfigureAfter和@AutoConfigureBefore等注解主要用于控制Spring组件的顺序，但它们也可以在一定程度上影响bean的加载顺序。







### **Spring循环依赖**

Spring循环依赖指的是两个或多个Bean之间相互依赖，形成一个环状依赖的情况。通俗的说，就是A依赖B，B依赖C，C依赖A，这样就形成了一个循环依赖的环。

 Spring循环依赖通常会导致Bean无法正确地被实例化，从而导致应用程序无法正常启动或者出现异常。因此，Spring循环依赖是一种需要尽量避免的情况



#### **循环依赖造成的原因**

**构造函数循环依赖**
在使用构造函数注入Bean时，如果两个Bean之间相互依赖，就可能会形成构造函数循环依赖，例如：

```java
@Component
public class A {    
    private B b;    
	public A(B b) {     
        this.b = b;   
    }}

@Component
public class B {   
    private A a;   
    public B(A a) {     
        this.a = a;  
    }}

```

上述代码，A、B的构造函数分别需要创建对方，A依赖B，B依赖A，它们之间形成了一个循环依赖。

当Spring容器启动时，它会尝试先实例化A，但是在实例化A的时候需要先实例化B，而实例化B的时候需要先实例化A，这样就形成了一个循环依赖的死循环，从而导致应用程序无法正常启动。


 **属性注入循环依赖**
 在使用属性注入Bean时，如果两个Bean之间相互依赖，就可能会形成属性循环依赖。例如：

```java
@Component
public class A {   
    @Autowired    
    private B b;

}

@Component
public class B {    
    @Autowired    
    private A a;
}
```

**对于构造器注入的循环依赖**，Spring处理不了，会直接抛出异常。我们可以使用@Lazy懒加载，什么时候需要对象再进行bean对象的创建

**对于属性注入的循环依赖**，是通过三级缓存处理来循环依赖的。







#### **三层缓存机制**

bean的创建流程

 依赖注入就发生在第二步，属性赋值，结合这个过程，Spring 通过三级缓存解决了循环依赖：

- 一级缓存 : Map<String,Object> **singletonObjects**，单例池，用于保存实例化、属性赋值（注入）、初始化完成的 bean 实例
- 二级缓存 : Map<String,Object> **earlySingletonObjects**，早期曝光对象，里面存放的只是进行了实例化的bean，还没有进行属性设置和初始化操作，也就是bean的创建还没有完成，还在进行中，这里是为了方便被别的bean引用
- 三级缓存 : Map<String,ObjectFactory<?>> **singletonFactories**，Spring中的每个bean创建都有自己专属的ObjectFactory工厂类，三级缓存缓存的就是对应的bean的工厂实例，可以通过该工厂实例的getObject()方法获取该bean的实例。



**三级缓存解决循环依赖的过程：**

**实例化A：**创建 A 实例，实例化的时候把 A 对象工厂放⼊三级缓存，表示 A 开始实例化了，虽然我这个对象还不完整，但是先曝光出来让大家知道

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240511171849766.png" alt="image-20240511171849766" style="zoom:50%;" />

**实例化B：**A 注注入属性时，发现依赖 B，此时 B 还没有被创建出来，所以去实例化 B

**填充A的引用到B：**同样，B 注入属性时发现依赖 A，它就会从缓存里找 A 对象。依次从一级到三级缓存查询 A，从三级缓存通过A的工厂拿到 A，发现 A 虽然不太完善，但是存在，把 A放入二级缓存（如果A有代理，则把A的代理放入），同时删除三级缓存中的 A

**完成B的创建：**此时，B 已经实例化并且初始化完成，把 B 放入一级缓存。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240511171529766.png" alt="image-20240511171529766" style="zoom: 50%;" />

**完成A的创建：**接着 A 继续属性赋值，顺利从⼀级缓存拿到实例化且初始化完成的 B 对象，A 对象创建也完成，删除二级缓存中的 A，同时把 A 放入一级缓存

**处理代理：**如果A有代理（例如，使用了Spring AOP），这个代理将在这个阶段被创建和应用。通常，代理涉及到创建一个新的对象（代理对象），它包装了原始的A实例，并提供了额外的功能（如方法拦截、事务管理等）。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240511172505838.png" alt="image-20240511172505838" style="zoom:50%;" />

最后，⼀级缓存中保存着实例化、初始化都完成的 A、B 对象

所以，我们就知道为什么 Spring 能解决 setter 注入的循环依赖了，因为实例化和属性赋值是分开的，所以里面有操作的空间。如果都是构造器注入的化，那么都得在实例化这一步完成注入，所以自然是无法支持了

> 注意：只有单例的 Bean 存在循环依赖的情况，Spring才可以解决，原型(Prototype)情况下，Spring 会直接抛出异常。







#### **为什么需要三级缓存而不是二级缓存？**

两级缓存分为两种情况来说，分别是 一级缓存 + 二级缓存 和 一级缓存 + 三级缓存 两种组合。

组合一：一级缓存 + 二级缓存

singletonObjects + earlySingletonObjects 理论可以解决依赖注入，也可以解决代理，但需要每次加入二级缓存都要是代理对象，如果没有代理就完全没有必要，同时也不符合 Spring 对 Bean 生命周期的定义。(对象都应该在创建建之后再进行动态代理而不是单纯的实例化以后就急着进行代理，如果循环依赖就是没办法的事)

组合二：一级缓存 + 三级缓存

singletonObjects + singletonFactories 可以解决依赖注入的问题，但是没法解决代理的问题，若要进行代理从 ObjectFactory 中获取对象实例进行代理，但是这样每次获取对象都不是同一个。**需要借助二级缓存保存半成品的代理对象，等到被代理对象创建完成，代理对象再完成创建**



### **Spring启动过程**

1. 读取web.xml文件。
2. 创建 ServletContext，为 ioc 容器提供宿主环境。
3. 触发容器初始化事件，调用 contextLoaderListener.contextInitialized()方法，在这个方法会初始化一个应用上下文WebApplicationContext，即 Spring 的 ioc 容器。ioc 容器初始化完成之后，会被存储到 ServletContext 中。
4. 初始化web.xml中配置的Servlet。如DispatcherServlet，用于匹配、处理每个servlet请求。










