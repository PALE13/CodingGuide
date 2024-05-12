## **MyBatis**



### **Mybatis是什么？**

- MyBatis框架是一个开源的数据持久层框架。
- 它的内部封装了通过JDBC访问数据库的操作，支持普通的SQL查询、存储过程和高级映射，几乎消除了所有的JDBC代码和参数的手工设置以及结果集的检索。
- MyBatis作为持久层框架，其主要思想是将程序中的大量SQL语句剥离出来，配置在配置文件当中，实现SQL的灵活配置。
- 这样做的好处是**将SQL与程序代码分离**，可以在不修改代码的情况下，直接在配置文件当中修改SQL。



### **MyBatis执行流程/工作原理**

①读取MyBatis配置文件：mybatis-config.xml加载运行环境和映射文件

②构造会话工厂SqlSessionFactory，一个项目只需要一个，单例的，一般由spring进行管理

③会话工厂创建SqlSession对象，这里面就含了执行SQL语句的所有方法

④操作数据库的接口，Executor执行器，同时负责查询缓存的维护

⑤Executor接口的执行方法中有一个MappedStatement类型的参数，封装了映射信息

⑥输入参数映射

⑦输出结果映射





### **MyBatis 是如何进行分页的**

**(1)** MyBatis 使用 RowBounds 对象进行逻辑分页，它是针对 ResultSet 结果集执行的内存分页，而非物理分页；

**(2)** 可以在 sql 内直接书写带有物理分页的参数来完成物理分页功能；

**(3)** 也可以使用分页插件来完成物理分页。

分页插件的基本原理是使用 MyBatis 提供的插件接口，实现自定义插件，在插件的拦截方法内拦截待执行的 sql，然后重写 sql，根据 dialect 方言，添加对应的物理分页语句和物理分页参数。

举例：`select _ from student` ，拦截 sql 后重写为：`select t._ from （select \* from student）t limit 0，10`





### **Mybatis缓存** 

MyBatis包含一个非常强大的查询缓存特性，它可以非常方便地定制和配置缓存。缓存可以极大的提升查询效率。

MyBatis系统中默认定义了两级缓存：一级缓存和二级缓存

- 默认情况下，只有一级缓存开启。（SqlSession级别的缓存，也称为本地缓存）
- 二级缓存需要手动开启和配置，他是基于namespace级别的缓存。
- 为了提高扩展性，MyBatis定义了缓存接口Cache，我们可以通过实现Cache接口来自定义二级缓存

#### **一级缓存**

一级缓存也叫本地缓存

- 与数据库同一次会话期间查询到的数据会放在本地缓存中。
- 以后如果需要获取相同的数据，直接从缓存中拿，没必须再去查询数据库
- 在多个SqlSession或者分布式环境下会产生脏读问题

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240512224938518.png" alt="image-20240512224938518" style="zoom: 67%;" />



**一级缓存失效的四种情况**

一级缓存是SqlSession级别的缓存，是一直开启的，我们关闭不了它

一级缓存失效情况：

- SqlSession不同
- SqlSession相同，查询条件不同
- SqlSession相同，两次查询之间执行了增删改操作**

- SqlSession相同，手动清除一级缓存






#### **二级缓存**

二级缓存也叫全局缓存，一级缓存作用域太低了，所以诞生了二级缓存，**可以被多个SqlSession共享，可以将粒度控制在namespace级别**

引入了CachingExecutor，是对Executor的装饰器模式，在执行Executor之前，会进行二级缓存的查询

查询流程是先查二级缓存，在查一级缓存，最后再查数据库

为了提高扩展性，MyBatis定义了缓存接口Cache，我们可以通过实现Cache接口来实现不同缓存实现类的组合

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240512225219713.png" alt="image-20240512225219713" style="zoom: 50%;" />





开启全局缓存 【mybatis-config.xml】

```java
<setting name="cacheEnabled" value="true"/>
```


去每个mapper.xml中配置使用二级缓存，这个配置非常简单；【xxxMapper.xml】

```
<cache/>
官方示例=====>查看官方文档
<cache
  eviction="FIFO"
  flushInterval="60000"
  size="512"
  readOnly="true"/>
```

这个更高级的配置创建了一个 FIFO 缓存，每隔 60 秒刷新，最多可以存储结果对象或列表的 512 个引用，而且返回的对象被认为是只读的，因此对它们进行修改可能会在不同线程中的调用者产生冲突。





**二级缓存清除方式**

1.LRU（Least Recently Used）策略：这种策略会清除最长时间未被使用的对象。当缓存达到最大容量时，最久未使用的数据会被自动移除

2.FIFO（First In First Out）策略：按照对象进入缓存的顺序进行清除，最早进入缓存的对象会被最先移除

3.软引用和弱引用策略：基于Java的软引用和弱引用特性，当JVM内存不足时，软引用缓存中的数据可能会被清除；而弱引用缓存中的数据在任何时候都可能被JVM清除

4.当执行更新操作（update、delete、insert）时：MyBatis会清空当前SqlSession的一级缓存。而对于二级缓存，它会在执行commit操作时被清空。这意味着，如果你在一个SqlSession中进行了增删改操作并提交了事务，那么不仅一级缓存会被清空，二级缓存也会受到影响







### **#{} 和 ${} 的区别是什么？**

- **`${}`是 Properties 文件中的变量占位符**，它可以用于标签属性值和 sql 内部，属于原样文本替换，可以替换任意内容，比如${driver}会被原样替换为`com.mysql.jdbc. Driver`。

**会将参数值直接拼接到 SQL 查询中，存在 SQL 注入的风险。**

一个示例：根据参数按任意字段排序：

```sql
select * from users order by ${orderCols}
```

`orderCols`可以是 `name`、`name desc`、`name,sex asc`等，实现灵活的排序。

- **`#{}`是 sql 的参数占位符**，MyBatis 会将 sql 中的`#{}`替换为? 号，在 sql 执行前会使用 PreparedStatement 的参数设置方法，按序给 sql 的? 号占位符设置参数值，比如 ps.setInt(0, parameterValue)，`#{item.name}` 的取值方式为使用反射从参数对象中获取 item 对象的 name 属性值，相当于 `param.getItem().getName()`。



### **xml 映射文件中的标签？**

答：还有很多其他的标签， `<resultMap>`、 `<parameterMap>`、 `<sql>`、 `<include>`、 `<selectKey>` ，加上动态 sql 的 9 个标签， `trim|where|set|foreach|if|choose|when|otherwise|bind` 等，其中 `<sql>` 为 sql 片段标签，通过 `<include>` 标签引入 sql 片段， `<selectKey>` 为不支持自增的主键生成策略标签。



### **Dao 接口的工作原理**

最佳实践中，通常一个 xml 映射文件，都会写一个 Dao 接口与之对应。Dao 接口就是人们常说的 `Mapper` 接口，接口的全限名，就是映射文件中的 namespace 的值，接口的方法名，就是映射文件中 `MappedStatement` 的 id 值，接口方法内的参数，就是传递给 sql 的参数。 `Mapper` 接口是没有实现类的，**当调用接口方法时，接口全限名+方法名拼接字符串作为 key 值，可唯一定位一个 `MappedStatement`** ，举例：`com.mybatis3.mappers. StudentDao.findStudentById` ，可以唯一找到 namespace 为 `com.mybatis3.mappers. StudentDao` 下面 `id = findStudentById` 的 `MappedStatement` 。在 MyBatis 中，每一个 `<select>`、 `<insert>`、 `<update>`、 `<delete>` 标签，都会被解析为一个 `MappedStatement` 对象。

**Dao 接口里的方法，参数不同时，方法能重载吗？**

Dao 接口里的方法可以重载，但是 Mybatis 的 xml 里面的 ID 不允许重复。

Mybatis 版本 3.3.0，亲测如下

```java
/**
 * Mapper接口里面方法重载
 */
public interface StuMapper {

 List<Student> getAllStu();

 List<Student> getAllStu(@Param("id") Integer id);
}
```

然后在 `StuMapper.xml` 中利用 Mybatis 的动态 sql 就可以实现。

```xml
<select id="getAllStu" resultType="com.pojo.Student">
  select * from student
  <where>
    <if test="id != null">
      id = #{id}
    </if>
  </where>
</select>
```

能正常运行，并能得到相应的结果，这样就实现了在 Dao 接口中写重载方法。

**Mybatis 的 Dao 接口可以有多个重载方法，但是多个接口对应的映射必须只有一个，否则启动会报错。**

相关 issue：[更正：Dao 接口里的方法可以重载，但是 Mybatis 的 xml 里面的 ID 不允许重复！open in new window](https://github.com/Snailclimb/JavaGuide/issues/1122)。

Dao 接口的工作原理是 JDK 动态代理，MyBatis 运行时会使用 JDK 动态代理为 Dao 接口生成代理 proxy 对象，代理对象 proxy 会拦截接口方法，转而执行 `MappedStatement` 所代表的 sql，然后将 sql 执行结果返回。

**补充**：Dao 接口方法可以重载，但是需要满足以下条件：

1. 仅有一个无参方法和一个有参方法
2. 多个有参方法时，参数数量必须一致。且使用相同的 `@Param` ，或者使用 `param1` 这种









###  **MyBatis 的插件运行原理**

MyBatis 仅可以编写针对 `ParameterHandler`、 `ResultSetHandler`、 `StatementHandler`、 `Executor` 这 4 种接口的插件，MyBatis 使用 JDK 的动态代理，为需要拦截的接口生成代理对象以实现接口方法拦截功能，每当执行这 4 种接口对象的方法时，就会进入拦截方法，具体就是 `InvocationHandler` 的 `invoke()` 方法，当然，只会拦截那些你指定需要拦截的方法。

实现 MyBatis 的 `Interceptor` 接口并复写 `intercept()` 方法，然后在给插件编写注解，指定要拦截哪一个接口的哪些方法即可，记住，别忘了在配置文件中配置你编写的插件。





### **MyBatis 动态 SQL**

MyBatis 动态 sql 可以让我们在 xml 映射文件内，以标签的形式编写动态 sql，完成逻辑判断和动态拼接 sql 的功能。其执行原理为，使用 OGNL 从 sql 参数对象中计算表达式的值，根据表达式的值动态拼接 sql，以此来完成动态 sql 的功能。

MyBatis 提供了 9 种动态 sql 标签:

- `<if></if>`
- `<where></where>(trim,set)`
- `<choose></choose>（when, otherwise）`
- `<foreach></foreach>`
- `<bind/>`

关于 MyBatis 动态 SQL 的详细介绍，请看这篇文章：[Mybatis 系列全解（八）：Mybatis 的 9 大动态 SQL 标签你知道几个？open in new window](https://segmentfault.com/a/1190000039335704) 。

关于这些动态 SQL 的具体使用方法，请看这篇文章：[Mybatis【13】-- Mybatis 动态 sql 标签怎么使用？](https://cloud.tencent.com/developer/article/1943349)



### **SQL注入问题**

假设有一个简单的登录功能，使用用户名和密码验证用户身份。以下是一个示例代码，其中存在 SQL 注入漏洞：

```java
javaCopy codeString username = request.getParameter("username");
String password = request.getParameter("password");

String sql = "SELECT * FROM users WHERE username='" + username + "' AND password='" + password + "'";
Statement statement = connection.createStatement();
ResultSet resultSet = statement.executeQuery(sql);
```

在上面的代码中，`username` 和 `password` 是通过用户输入获取的。假设攻击者输入的用户名是 `' OR '1'='1'--`，密码是任意值。那么经过拼接后的 SQL 查询语句将变为：

```java
SELECT * FROM users WHERE username='' OR '1'='1'--' AND password='任意值'
```

这条 SQL 语句的含义是查询用户名为空（`username=''`），或者 `1=1`（始终成立的条件），然后注释掉后面的内容（`--` 后面的部分被注释掉）。这样，攻击者可以绕过正常的用户名和密码验证，成功登录系统。

**如何防止SQL注入**

要防止 SQL 注入攻击，**应该使用参数化查询或者预处理语句**，而不是直接拼接用户输入的字符串到 SQL 查询中。以下是改写后的代码示例：

```java
javaCopy codeString username = request.getParameter("username");
String password = request.getParameter("password");

String sql = "SELECT * FROM users WHERE username=? AND password=?";
PreparedStatement statement = connection.prepareStatement(sql);
statement.setString(1, username);
statement.setString(2, password);
ResultSet resultSet = statement.executeQuery();
```

在上面的代码中，我们使用了 `PreparedStatement` 对象来构建 SQL 查询，其中的占位符 `?` 表示参数位置。然后使用 `setString` 方法将参数值传递给占位符，这样就避免了直接拼接用户输入的字符串到 SQL 查询中。这种方式可以有效地防止 SQL 注入攻击，因为数据库会将参数值视为数据，而不是 SQL 代码。





### **MyBatis 是如何将 sql 执行结果封装为目标对象并返回的**

第一种是**使用 `<resultMap>` 标签**，逐一定义列名和对象属性名之间的映射关系。

第二种是使用 sql 列的别名功能，将列别名书写为对象属性名，比如 T_NAME AS NAME，对象属性名一般是 name，小写，但是列名不区分大小写，MyBatis 会忽略列名大小写，智能找到与之对应对象属性名，你甚至可以写成 T_NAME AS NaMe，MyBatis 一样可以正常工作。

**有了列名与属性名的映射关系后，MyBatis 通过反射创建对象，同时使用反射给对象的属性逐一赋值并返回**，那些找不到映射关系的属性，是无法完成赋值的。





### **MyBatis 一对一、一对多的关联查询**

MyBatis 不仅可以执行一对一、一对多的关联查询，还可以执行多对一，多对多的关联查询，多对一查询，其实就是一对一查询，只需要把 `selectOne()` 修改为 `selectList()` 即可；多对多查询，其实就是一对多查询，只需要把 `selectOne()` 修改为 `selectList()` 即可。

关联对象查询，有两种实现方式，一种是单独发送一个 sql 去查询关联对象，赋给主对象，然后返回主对象。另一种是使用嵌套查询，嵌套查询的含义为使用 join 查询，一部分列是 A 对象的属性值，另外一部分列是关联对象 B 的属性值，好处是只发一个 sql 查询，就可以把主对象和其关联对象查出来。

那么问题来了，join 查询出来 100 条记录，如何确定主对象是 5 个，而不是 100 个？其去重复的原理是 `<resultMap>` 标签内的 `<id>` 子标签，指定了唯一确定一条记录的 id 列，MyBatis 根据 `<id>` 列值来完成 100 条记录的去重复功能， `<id>` 可以有多个，代表了联合主键的语意。

同样主对象的关联对象，也是根据这个原理去重复的，尽管一般情况下，只有主对象会有重复记录，关联对象一般不会重复。

举例：下面 join 查询出来 6 条记录，一、二列是 Teacher 对象列，第三列为 Student 对象列，MyBatis 去重复处理后，结果为 1 个老师 6 个学生，而不是 6 个老师 6 个学生。

| t_id | t_name  | s_id |
| ---- | ------- | ---- |
| 1    | teacher | 38   |
| 1    | teacher | 39   |
| 1    | teacher | 40   |
| 1    | teacher | 41   |
| 1    | teacher | 42   |
| 1    | teacher | 43   |





### **MyBatis 的延迟加载**

MyBatis 仅支持 association 关联对象和 collection 关联集合对象的延迟加载，association 指的就是一对一，collection 指的就是一对多查询。在 MyBatis 配置文件中，可以配置是否启用延迟加载 `lazyLoadingEnabled=true|false。`

**它的原理是，使用 `CGLIB` 创建目标对象的代理对象**，当调用目标方法时，进入拦截器方法，比如调用 `a.getB().getName()` ，拦截器 `invoke()` 方法发现 `a.getB()` 是 null 值，那么就会单独发送事先保存好的查询关联 B 对象的 sql，把 B 查询上来，然后调用 a.setB(b)，于是 a 的对象 b 属性就有值了，接着完成 `a.getB().getName()` 方法的调用。这就是延迟加载的基本原理。

当然了，不光是 MyBatis，几乎所有的包括 Hibernate，支持延迟加载的原理都是一样的。





### **MyBatis 的 xml 映射文件中，不同的 xml 映射文件，id 是否可以重复？**

答：不同的 xml 映射文件，如果配置了 namespace，那么 id 可以重复；如果没有配置 namespace，那么 id 不能重复；毕竟 namespace 不是必须的，只是最佳实践而已。

原因就是 namespace+id 是作为 `Map<String, MappedStatement>` 的 key 使用的，如果没有 namespace，就剩下 id，那么，id 重复会导致数据互相覆盖。有了 namespace，自然 id 就可以重复，namespace 不同，namespace+id 自然也就不同。



### **MyBatis 都有哪些 Executor 执行器**

MyBatis 有三种基本的 `Executor` 执行器：

- **`SimpleExecutor`：** 每执行一次 update 或 select，就开启一个 Statement 对象，用完立刻关闭 Statement 对象。
- **`ReuseExecutor`：** 执行 update 或 select，以 sql 作为 key 查找 Statement 对象，存在就使用，不存在就创建，用完后，不关闭 Statement 对象，而是放置于 Map<String, Statement>内，供下一次使用。简言之，就是重复使用 Statement 对象。
- **`BatchExecutor`**：执行 update（没有 select，JDBC 批处理不支持 select），将所有 sql 都添加到批处理中（addBatch()），等待统一执行（executeBatch()），它缓存了多个 Statement 对象，每个 Statement 对象都是 addBatch()完毕后，等待逐一执行 executeBatch()批处理。与 JDBC 批处理相同。

作用范围：`Executor` 的这些特点，都严格限制在 SqlSession 生命周期范围内。









### **简述 MyBatis 的 xml 映射文件和 MyBatis 内部数据结构之间的映射关系？**

MyBatis 将所有 xml 配置信息都封装到 All-In-One 重量级对象 Configuration 内部。在 xml 映射文件中， `<parameterMap>` 标签会被解析为 `ParameterMap` 对象，其每个子元素会被解析为 ParameterMapping 对象。 `<resultMap>` 标签会被解析为 `ResultMap` 对象，其每个子元素会被解析为 `ResultMapping` 对象。每一个 `<select>、<insert>、<update>、<delete>` 标签均会被解析为 `MappedStatement` 对象，标签内的 sql 会被解析为 BoundSql 对象。





### **为什么MyBatis 是半自动 ORM 映射工具**

Hibernate 属于全自动 ORM 映射工具，使用 Hibernate 查询关联对象或者关联集合对象时，可以根据对象关系模型直接获取，所以它是全自动的。而 MyBatis 在查询关联对象或关联集合对象时，需要手动编写 sql 来完成，所以，称之为半自动 ORM 映射工具。











