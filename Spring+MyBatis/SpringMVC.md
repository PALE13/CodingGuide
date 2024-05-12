## SpringMVC



#### **说一下对MVC的理解**

 MVC是一种设计模式,在这种模式下软件被分为三层,即Model（模型）、View（视图）、Controller（控制器）。Model代表的是数据,View代表的是用户界面,Controller代表的是数据的处理逻辑,它是Model和View这两层的桥梁。

将软件分层的好处是,，可以将对象之间的耦合度降低，便于代码的维护。 

Model：指从现实世界中抽象出来的对象模型，是应用逻辑的反应；它封装了数据和对数据的操作，是实际进行数据处理的地方（模型层与数据库才有交互）。在MVC的三个部件中，模型拥有最多的处理任务。被模型返回的数据是中立的，模型与数据格式无关，这样一个模型能为多个视图提供数据，由于应用于模型的代码只需写一次就可以被多个视图重用，所以减少了代码的重复性。 

View：负责进行模型的展示,一般就是我们见到的用户界面。 

Controller：控制器负责视图和模型之间的交互,控制对用户输入的响应、响应方式和流程；它主要负责两方面的动作,一是把用户的请求分发到相应的模型，二是把模型的改变及时地反映到视图上。 

为了解耦以及提升代码的可维护性，服务端开发一般会对代码进行分层，服务端代码一般会分为三层：表现层、业务层、数据访问层。在浏览器访问服务器时，请求会先到达表现层，最典型的MVC就是jsp+servlet+javabean模式。 以JavaBean作为模型,既可以作为数据模型来封装业务数据,又可以作为业务逻辑模型来包含应用的业务操作。 

JSP作为视图层，负责提供页面为用户展示数据，提供相应的表单（Form）来用于用户的请求，并在适当的时候（点击按钮）向控制器发出请求来请求模型进行更新。 Serlvet作为控制器，用来接收用户提交的请求，然后获取请求中的数据，将之转换为业务模型需要的数据模型，然后调用业务模型相应的业务方法进行更新，同时根据业务执行结果来选择要返回的视图。 当然,这种方式现在已经不那么流行了，Spring MVC框架已经成为了MVC模式的最主流实现。 

Spring MVC框架是基于Java的实现了MVC框架模式的请求驱动类型的轻量级框架。前端控制器是DispatcherServlet接口实现类,映射处理器是HandlerMapping接口实现类,视图解析器是ViewResolver接口实现类,页面控制器是Controller接口实现类。



#### **说说自己对于 Spring MVC 了解?**

MVC 是模型(Model)、视图(View)、控制器(Controller)的简写，其核心思想是通过将业务逻辑、数据、显示分离来组织代码。

![img](https://oss.javaguide.cn/java-guide-blog/image-20210809181452421.png)

网上有很多人说 MVC 不是设计模式，只是软件设计规范，我个人更倾向于 MVC 同样是众多设计模式中的一种。

**Model 1 时代**

很多学 Java 后端比较晚的朋友可能并没有接触过 Model 1 时代下的 JavaWeb 应用开发。在 Model1 模式下，整个 Web 应用几乎全部用 JSP 页面组成，只用少量的 JavaBean 来处理数据库连接、访问等操作。

这个模式下 JSP 即是控制层（Controller）又是表现层（View）。显而易见，这种模式存在很多问题。比如控制逻辑和表现逻辑混杂在一起，导致代码重用率极低；再比如前端和后端相互依赖，难以进行测试维护并且开发效率极低。

<img src="https://oss.javaguide.cn/java-guide-blog/mvc-mode1.png" alt="mvc-mode1" style="zoom: 80%;" />

**Model 2 时代**

学过 Servlet 并做过相关 Demo 的朋友应该了解“Java Bean(Model)+ JSP（View）+Servlet（Controller） ”这种开发模式，这就是早期的 JavaWeb MVC 开发模式。

- Model:系统涉及的数据，也就是 dao 和 bean。
- View：展示模型中的数据，只是用来展示。
- Controller：接受用户请求，并将请求发送至 Model，最后返回数据给 JSP 并展示给用户

<img src="https://oss.javaguide.cn/java-guide-blog/mvc-model2.png" alt="img" style="zoom: 80%;" />

Model2 模式下还存在很多问题，Model2 的抽象和封装程度还远远不够，使用 Model2 进行开发时不可避免地会重复造轮子，这就大大降低了程序的可维护性和复用性。

于是，很多 JavaWeb 开发相关的 MVC 框架应运而生比如 Struts2，但是 Struts2 比较笨重。



**Spring MVC 时代**

随着 Spring 轻量级开发框架的流行，Spring 生态圈出现了 Spring MVC 框架， Spring MVC 是当前最优秀的 MVC 框架。相比于 Struts2 ， Spring MVC 使用更加简单和方便，开发效率更高，并且 Spring MVC 运行速度更快。

MVC 是一种设计模式，Spring MVC 是一款很优秀的 MVC 框架。Spring MVC 可以帮助我们进行更简洁的 Web 层的开发，并且它天生与 Spring 框架集成。Spring MVC 下我们一般把后端项目分为 Service 层（处理业务）、Dao 层（数据库操作）、Entity 层（实体类）、Controller 层(控制层，返回数据给前台页面)。



#### **Spring MVC 的核心组件有哪些？**

记住了下面这些组件，也就记住了 SpringMVC 的工作原理。

- **`DispatcherServlet`**：**核心的中央处理器**，负责接收请求、分发，并给予客户端响应。
- **`HandlerMapping`**：**处理器映射器**，根据 URL 去匹配查找能处理的 `Handler` ，并会将请求涉及到的拦截器和 `Handler` 一起封装。
- **`HandlerAdapter`**：**处理器适配器**，根据 `HandlerMapping` 找到的 `Handler` ，适配执行对应的 `Handler`；
- **`Handler`**：**请求处理器**，处理实际请求的处理器。
- **`ViewResolver`**：**视图解析器**，根据 `Handler` 返回的逻辑视图 / 视图，解析并渲染真正的视图，并传递给 `DispatcherServlet` 响应客户端



#### **SpringMVC 工作原理了解吗?**

**Spring MVC 原理如下图所示：**

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240312125953988.png" alt="image-20240312125953988" style="zoom:67%;" />



**流程说明（重要）：**

1. 客户端（浏览器）发送请求， `DispatcherServlet`拦截请求。
2. `DispatcherServlet` 根据请求信息调用 `HandlerMapping` **。`HandlerMapping` 根据 URL 去匹配查找能处理的 `Handler`（也就是我们平常说的 `Controller` 控制器）** ，并会将请求涉及到的拦截器和 `Handler` 一起封装。
3. `DispatcherServlet` 调用 `HandlerAdapter`适配器执行 `Handler` 。
4. `Handler` 完成对用户请求的处理后，会返回一个 `ModelAndView` 对象给`DispatcherServlet`，`ModelAndView` 顾名思义，包含了数据模型以及相应的视图的信息。`Model` 是返回的数据对象，`View` 是个逻辑上的 `View`。
5. `ViewResolver` 会根据逻辑 `View` 查找实际的 `View`。
6. `DispaterServlet` 把返回的 `Model` 传给 `View`（视图渲染）。
7. 把 `View` 返回给请求者（浏览器）











