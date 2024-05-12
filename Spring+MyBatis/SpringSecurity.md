## Spring Security

Spring Security 重要的是实战，这里仅对小部分知识点进行总结。

### @PreAuthorize有哪些控制请求访问权限的方法？

@PreAuthorize是Spring Security中的一个注解，用于在方法执行前对访问权限进行验证。它基于SpEL（Spring Expression Language）表达式，可以在方法级别对用户的角色、权限或其他条件进行验证，以确定是否允许执行该方法。

`permitAll()`：无条件允许任何形式访问，不管你登录还是没有登录。

`anonymous()`：允许匿名访问，也就是没有登录才可以访问。

`denyAll()`：无条件决绝任何形式的访问。

`authenticated()`：只允许已认证的用户访问。

`fullyAuthenticated()`：只允许已经登录或者通过 remember-me 登录的用户访问。

`hasRole(String)` : 只允许指定的角色访问。

`hasAnyRole(String)` : 指定一个或者多个角色，满足其一的用户即可访问。

`hasAuthority(String)`：只允许具有指定权限的用户访问

`hasAnyAuthority(String)`：指定一个或者多个权限，满足其一的用户即可访问。

`hasIpAddress(String)` : 只允许指定 ip 的用户访问。





### hasRole 和 hasAuthority 有区别吗？

在Spring Security中，hasRole和hasAuthority是两个用于访问控制的表达式方法，虽然它们的功能有一些相似之处，但在使用和语义上存在一些区别。

hasRole：hasRole方法用于检查用户是否具有指定的角色。角色在Spring Security中是以"ROLE_"前缀开头的字符串表示，例如"ROLE_ADMIN"、"ROLE_USER"等。hasRole方法会自动添加"ROLE_"前缀，因此在使用时可以省略该前缀。通常情况下，hasRole用于基于角色的访问控制。

示例：

```java
@PreAuthorize("hasRole('ROLE_ADMIN')")
public void adminOnlyMethod() {
    // 只有具有 "ROLE_ADMIN" 角色的用户可以访问该方法
}
```


hasAuthority：hasAuthority方法用于检查用户是否具有指定的权限。权限可以是任意字符串，表示用户在系统中具有的某种权限或许可。hasAuthority方法根据权限字符串进行匹配，不会自动添加前缀。通常情况下，hasAuthority用于细粒度的访问控制，更灵活。

示例：

```java
@PreAuthorize("hasAuthority('WRITE_ARTICLE')")
public void writeArticle() {
    // 只有具有 "WRITE_ARTICLE" 权限的用户可以访问该方法
}
```


authority 描述的的是一个具体的权限，例如针对某一项数据的查询或者删除权限，它是一个 permission，例如 read_employee、delete_employee、update_employee 之类的







### 如何对密码进行加密？

如果我们需要保存密码这类敏感数据到数据库的话，需要先加密再保存。

Spring Security 提供了多种加密算法的实现，开箱即用，非常方便。这些加密算法实现类的父类是 `PasswordEncoder` ，如果你想要自己实现一个加密算法的话，也需要继承 `PasswordEncoder`。

`PasswordEncoder` 接口一共也就 3 个必须实现的方法。

```java
public interface PasswordEncoder {
    // 加密也就是对原始密码进行编码
    String encode(CharSequence var1);
    // 比对原始密码和数据库中保存的密码
    boolean matches(CharSequence var1, String var2);
    // 判断加密密码是否需要再次进行加密，默认返回 false
    default boolean upgradeEncoding(String encodedPassword) {
        return false;
    }
}
```

**多种实现类**

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240319182509640.png" alt="image-20240319182509640" style="zoom: 80%;" />

官方推荐使用基于 bcrypt 强哈希函数的加密算法实现类。





#### **如何优雅更换系统使用的加密算法？**

如果我们在开发过程中，突然发现现有的加密算法无法满足我们的需求，需要更换成另外一个加密算法，这个时候应该怎么办呢？

推荐的做法是通过 `DelegatingPasswordEncoder` 兼容多种不同的密码加密方案，以适应不同的业务需求。

从名字也能看出来，`DelegatingPasswordEncoder` 其实就是一个代理类，并非是一种全新的加密算法，它做的事情就是代理上面提到的加密算法实现类。在 Spring Security 5.0 之后，默认就是基于 `DelegatingPasswordEncoder` 进行密码加密的。

下面是使用DelegatingPasswordEncoder的一般步骤：

创建DelegatingPasswordEncoder实例：使用DelegatingPasswordEncoder的构造函数创建实例，并设置默认的密码编码器。

```java
DelegatingPasswordEncoder passwordEncoder = new DelegatingPasswordEncoder("bcrypt");
```

在上述示例中，我们创建了一个DelegatingPasswordEncoder实例，并设置了默认的密码编码器为"bcrypt"。

在上述示例中，我们创建了一个DelegatingPasswordEncoder实例，并设置了默认的密码编码器为"bcrypt"。

注册其他密码编码器：根据需要，可以注册其他密码编码器到DelegatingPasswordEncoder中，以支持多种密码编码算法。

```java
passwordEncoder.setDefaultPasswordEncoderForMatches(new BCryptPasswordEncoder());
passwordEncoder.addPasswordEncoder(new NoOpPasswordEncoder.getInstance(), "plain");
```

在上述示例中，我们注册了BCryptPasswordEncoder作为"bcrypt"算法的密码编码器，并注册了NoOpPasswordEncoder作为"plain"算法的密码编码器。

在上述示例中，我们注册了BCryptPasswordEncoder作为"bcrypt"算法的密码编码器，并注册了NoOpPasswordEncoder作为"plain"算法的密码编码器。

加密密码：使用DelegatingPasswordEncoder的encode()方法对密码进行加密。

```java
String rawPassword = "myPassword";
boolean matches = passwordEncoder.matches(rawPassword, encodedPassword);
```


通过使用DelegatingPasswordEncoder，您可以根据密码的前缀选择适当的密码编码器，从而支持多种密码编码算法，并实现密码的加密和验证功能。