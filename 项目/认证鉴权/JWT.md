### 什么是 JWT?

JWT （JSON Web Token） 是目前最流行的跨域认证解决方案，**是一种基于 Token 的认证授权机制**。 从 JWT 的全称可以看出，JWT 本身也是 Token，一种规范化之后的 JSON 结构的 Token。

JWT 自身包含了身份验证所需要的所有信息，因此，我们的服务器不需要存储 Session 信息。这显然增加了系统的可用性和伸缩性，大大减轻了服务端的压力。

可以看出，**JWT 更符合设计 RESTful API 时的「Stateless（无状态）」原则** 。

并且， 使用 JWT 认证可以有效避免 CSRF 攻击，**因为 JWT 一般是存在 localStorage 中**，使用 JWT 进行身份验证的过程中是不会涉及到 Cookie 的。





JWT 本质上就是一组字串，通过（`.`）切分成三个为 Base64 编码的部分：

- **Header** : 描述 JWT 的元数据，定义了生成签名的算法以及 `Token` 的类型。
- **Payload** : 用来存放实际需要传递的数据
- **Signature（签名）**：服务器通过 Payload、Header 和一个密钥(Secret)使用 Header 里面指定的签名算法（默认是 HMAC SHA256）生成。

JWT 通常是这样的：`xxxxx.yyyyy.zzzzz`。

示例：

```plain
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```



**Header**

Header 通常由两部分组成：

- `typ`（Type）：令牌类型，也就是 JWT。
- `alg`（Algorithm）：签名算法，比如 HS256。

示例：

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

JSON 形式的 Header 被转换成 Base64 编码，成为 JWT 的第一部分。



**Payload**

Payload 也是 JSON 格式数据，其中包含了 Claims(声明，包含 JWT 的相关信息)。

Claims 分为三种类型：

- **Registered Claims（注册声明）**：预定义的一些声明，建议使用，但不是强制性的。
- **Public Claims（公有声明）**：JWT 签发方可以自定义的声明，但是为了避免冲突，应该在 [IANA JSON Web Token Registryopen in new window](https://www.iana.org/assignments/jwt/jwt.xhtml) 中定义它们。
- **Private Claims（私有声明）**：JWT 签发方因为项目需要而自定义的声明，更符合实际项目场景使用。

下面是一些常见的注册声明：

- `iss`（issuer）：JWT 签发方。
- `iat`（issued at time）：JWT 签发时间。
- `sub`（subject）：JWT 主题。
- `aud`（audience）：JWT 接收方。
- `exp`（expiration time）：JWT 的过期时间。
- `nbf`（not before time）：JWT 生效时间，早于该定义的时间的 JWT 不能被接受处理。
- `jti`（JWT ID）：JWT 唯一标识。

示例：

```json
{
  "uid": "ff1212f5-d8d1-4496-bf41-d2dda73de19a",
  "sub": "1234567890",
  "name": "John Doe",
  "exp": 15323232,
  "iat": 1516239022,
  "scope": ["admin", "user"]
}
```

Payload 部分默认是不加密的，**一定不要将隐私信息存放在 Payload 当中！！！**

JSON 形式的 Payload 被转换成 Base64 编码，成为 JWT 的第二部分。



**Signature**

Signature 部分是对前两部分的签名，作用是防止 JWT（主要是 payload） 被篡改。

这个签名的生成需要用到：

- Header + Payload。
- 存放在服务端的密钥。
- 签名算法。

签名的计算公式如下：

```plain
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

算出签名以后，把 Header、Payload、Signature 三个部分拼成一个字符串，每个部分之间用"点"（`.`）分隔，这个字符串就是 JWT 



### 如何基于 JWT 进行身份验证？

在基于 JWT 进行身份验证的的应用程序中，服务器通过 Payload、Header 和 Secret(密钥)创建 JWT 并将 JWT 发送给客户端。客户端接收到 JWT 之后，会将其保存在 Cookie 或者 localStorage 里面，以后客户端发出的所有请求都会携带这个令牌。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240415113403644.png" alt="image-20240415113403644" style="zoom: 67%;" />

简化后的步骤如下：

1. 用户向服务器发送用户名、密码以及验证码用于登陆系统。
2. 如果用户用户名、密码以及验证码校验正确的话，服务端会返回已经签名的 Token，也就是 JWT。
3. 用户以后每次向后端发请求都在 Header 中带上这个 JWT 。
4. 服务端检查 JWT 并从中获取用户相关信息。

两点建议：

1. 建议将 JWT 存放在 localStorage 中，放在 Cookie 中会有 CSRF 风险。
2. 请求服务端并携带 JWT 的常见做法是将其放在 HTTP Header 的 `Authorization` 字段中（`Authorization: Bearer Token`）。

**[spring-security-jwt-guideopen in new window](https://github.com/Snailclimb/spring-security-jwt-guide)** 就是一个基于 JWT 来做身份认证的简单案例，感兴趣的可以看看。



### 如何防止 JWT 被篡改？

有了签名之后，即使 JWT 被泄露或者截获，黑客也没办法同时篡改 Signature、Header、Payload。

这是为什么呢？因为服务端拿到 JWT 之后，会解析出其中包含的 Header、Payload 以及 Signature 。服务端会根据 Header、Payload、密钥再次生成一个 Signature。拿新生成的 Signature 和 JWT 中的 Signature 作对比，如果一样就说明 Header 和 Payload 没有被修改。

不过，如果服务端的秘钥也被泄露的话，黑客就可以同时篡改 Signature、Header、Payload 了。黑客直接修改了 Header 和 Payload 之后，再重新生成一个 Signature 就可以了。

**密钥一定保管好，一定不要泄露出去。JWT 安全的核心在于签名，签名安全的核心在密钥。**





### **为什么JWT比Session-Cookie更好**

#### **安全性**

**同源策略：** LocalStorage 受到同源策略的限制，即只有同源的页面才能访问相同的 LocalStorage 数据。这意味着其他域名下的脚本代码无法直接访问当前域名下的 LocalStorage 数据，从而降低了跨站点攻击的风险。

一般情况下我们使用 JWT 的话，在我们登录成功获得 JWT 之后，一般会选择存放在 localStorage 中。前端的每一个请求后续都会附带上这个 JWT，整个过程压根不会涉及到 Cookie。因此，**即使你点击了非法链接发送了请求到服务端，这个非法请求也是不会携带 JWT 的，所以这个请求将是非法的。**

**HTTP 请求不携带：** LocalStorage 的数据不会随着每次 HTTP 请求自动发送到服务器，只有在客户端 JavaScript 显式获取并发送的情况下才可能发送。这相比 Cookie 在每次请求中都会自动发送的特性，降低了数据被盗取的可能性。



尽管 LocalStorage 有上述优势，但仍然存在一些潜在的安全风险：

1. **XSS 攻击：** 跨站脚本攻击（Cross-Site Scripting，XSS）是一种常见的 Web 安全漏洞，攻击者通过注入恶意脚本代码来获取用户的敏感信息。如果网站存在 XSS 漏洞，攻击者可以通过 JavaScript 获取到 LocalStorage 中的数据，包括 JWT。
2. **本地文件访问：** 在某些情况下，本地文件（file://）也可以访问 LocalStorage 中的数据。虽然这种情况较少见，但在一些特殊场景下可能会存在风险。
3. **本地劫持：** 如果客户端的设备受到恶意软件或者攻击者控制，可能会导致本地存储被劫持，从而获取到 LocalStorage 中的数据。

#### **适合移动端应用**

使用 Session 进行身份认证的话，需要保存一份信息在服务器端，而且这种方式会依赖到 Cookie（需要 Cookie 保存 `SessionId`），所以不适合移动端。

但是，使用 JWT 进行身份认证就不会存在这种问题，因为只要 JWT 可以被客户端存储就能够使用，而且 JWT 还可以跨语言使用。

> 为什么使用 Session 进行身份认证的话不适合移动端 ？
>
> 1. 状态管理: Session 基于服务器端的状态管理，而移动端应用通常是无状态的。移动设备的连接可能不稳定或中断，因此难以维护长期的会话状态。如果使用 Session 进行身份认证，移动应用需要频繁地与服务器进行会话维护，增加了网络开销和复杂性;
> 2. 兼容性: 移动端应用通常会面向多个平台，如 iOS、Android 和 Web。每个平台对于 Session 的管理和存储方式可能不同，可能导致跨平台兼容性的问题;
> 3. 安全性: 移动设备通常处于不受信任的网络环境，存在数据泄露和攻击的风险。将敏感的会话信息存储在移动设备上增加了被攻击的潜在风险。

#### **单点登录友好**

使用 Session 进行身份认证的话，实现单点登录，需要我们把用户的 Session 信息保存在一台电脑上，并且还会遇到常见的 Cookie 跨域的问题。但是，使用 JWT 进行认证的话， JWT 被保存在客户端，不会存在这些问题。

------

著作权归JavaGuide(javaguide.cn)所有 基于MIT协议 原文链接：https://javaguide.cn/system-design/security/advantages-and-disadvantages-of-jwt.html







### 如何加强 JWT 的安全性？

1. 使用安全系数高的加密算法。
2. 使用成熟的开源库，没必要造轮子。
3. JWT 存放在 localStorage 中而不是 Cookie 中，避免 CSRF 风险。
4. 一定不要将隐私信息存放在 Payload 当中。
5. 密钥一定保管好，一定不要泄露出去。JWT 安全的核心在于签名，签名安全的核心在密钥。
6. Payload 要加入 `exp` （JWT 的过期时间），永久有效的 JWT 不合理。并且，JWT 的过期时间不易过长。



### JWT 身份认证常见问题及解决办法

与之类似的具体相关场景有：

- 退出登录;
- 修改密码;
- 服务端修改了某个用户具有的权限或者角色；
- 用户的帐户被封禁/删除；
- 用户被服务端强制注销；
- 用户被踢下线；

这个问题不存在于 Session 认证方式中，因为在 Session 认证方式中，遇到这种情况的话服务端删除对应的 Session 记录即可。但是，使用 JWT 认证的方式就不好解决了。我们也说过了，JWT 一旦派发出去，如果后端不增加其他逻辑的话，它在失效之前都是有效的



**1、将 JWT 存入数据库**

将有效的 JWT 存入数据库中，更建议使用内存数据库比如 Redis。如果需要让某个 JWT 失效就直接从 Redis 中删除这个 JWT 即可。但是，这样会导致每次使用 JWT 都要先从 Redis 中查询 JWT 是否存在的步骤，而且违背了 JWT 的无状态原则。

**2、黑名单机制**

和上面的方式类似，使用内存数据库比如 Redis 维护一个黑名单，如果想让某个 JWT 失效的话就直接将这个 JWT 加入到 **黑名单** 即可。然后，每次使用 JWT 进行请求的话都会先判断这个 JWT 是否存在于黑名单中。

前两种方案的核心在于将有效的 JWT 存储起来或者将指定的 JWT 拉入黑名单。

虽然这两种方案都违背了 JWT 的无状态原则，但是一般实际项目中我们通常还是会使用这两种方案。



**3、修改密钥 (Secret)** :

我们为每个用户都创建一个专属密钥，如果我们想让某个 JWT 失效，我们直接修改对应用户的密钥即可。但是，这样相比于前两种引入内存数据库带来了危害更大：

- 如果服务是分布式的，则每次发出新的 JWT 时都必须在多台机器同步密钥。为此，你需要将密钥存储在数据库或其他外部服务中，这样和 Session 认证就没太大区别了。
- 如果用户同时在两个浏览器打开系统，或者在手机端也打开了系统，如果它从一个地方将账号退出，那么其他地方都要重新进行登录，这是不可取的。

**4、保持令牌的有效期限短并经常轮换**

很简单的一种方式。但是，会导致用户登录状态不会被持久记录，而且需要用户经常登录。

另外，对于修改密码后 JWT 还有效问题的解决还是比较容易的。说一种我觉得比较好的方式：**使用用户的密码的哈希值对 JWT 进行签名。因此，如果密码更改，则任何先前的令牌将自动无法验证。**





### **JWT 的续签问题**

JWT 有效期一般都建议设置的不太长，那么 JWT 过期后如何认证，如何实现动态刷新 JWT，避免用户经常需要重新登录？

我们先来看看在 Session 认证中一般的做法：**假如 Session 的有效期 30 分钟，如果 30 分钟内用户有访问，就把 Session 有效期延长 30 分钟。**

JWT 认证的话，我们应该如何解决续签问题呢？查阅了很多资料，我简单总结了下面 4 种方案：

**1、类似于 Session 认证中的做法（不推荐）**

这种方案满足于大部分场景。假设服务端给的 JWT 有效期设置为 30 分钟，服务端每次进行校验时，如果发现 JWT 的有效期马上快过期了，服务端就重新生成 JWT 给客户端。客户端每次请求都检查新旧 JWT，如果不一致，则更新本地的 JWT。这种做法的问题是仅仅在快过期的时候请求才会更新 JWT ，对客户端不是很友好。

**2、每次请求都返回新 JWT（不推荐）**

这种方案的的思路很简单，但是，开销会比较大，尤其是在服务端要存储维护 JWT 的情况下。

**3、JWT 有效期设置到半夜（不推荐）**

这种方案是一种折衷的方案，保证了大部分用户白天可以正常登录，适用于对安全性要求不高的系统。

**4、用户登录返回两个 JWT（推荐）**

第一个是 accessJWT ，它的过期时间 JWT 本身的过期时间比如半个小时，另外一个是 refreshJWT 它的过期时间更长一点比如为 1 天。refreshJWT 只用来获取 accessJWT，不容易被泄露。

客户端登录后，将 accessJWT 和 refreshJWT 保存在本地，每次访问将 accessJWT 传给服务端。服务端校验 accessJWT 的有效性，如果过期的话，就将 refreshJWT 传给服务端。如果有效，服务端就生成新的 accessJWT 给客户端。否则，客户端就重新登录即可。

这种方案的不足是：

- 需要客户端来配合；
- 用户注销的时候需要同时保证两个 JWT 都无效；
- 重新请求获取 JWT 的过程中会有短暂 JWT 不可用的情况（可以通过在客户端设置定时器，当 accessJWT 快过期的时候，提前去通过 refreshJWT 获取新的 accessJWT）;
- 存在安全问题，只要拿到了未过期的 refreshJWT 就一直可以获取到 accessJWT。不过，由于 refreshJWT 只用来获取 accessJWT，不容易被泄露。













