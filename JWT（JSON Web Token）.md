# JWT（JSON Web Token）

## 一、Session、Cookie与JWT的区别

### 1.1 Cookie 简介

- HTTP 是无状态的协议（对于事务处理没有记忆能力，每次客户端和服务端会话完成时，服务端不会保存任何会话信息）：每个请求都是完全独立的，服务端无法确认当前访问者的身份信息，无法分辨上一次的请求发送者和这一次的发送者是不是同一个人。所以服务器与浏览器为了进行会话跟踪（知道是谁在访问我），就必须主动的去维护一个状态，这个状态用于告知服务端前后两个请求是否来自同一浏览器。而这个状态需要通过 cookie 或者 session 去实现。
- cookie 存储在客户端： cookie 是服务器发送到用户浏览器并保存在本地的一小块数据，它会在浏览器下次向同一服务器再发起请求时被携带并发送到服务器上。
- cookie 是不可跨域的： 每个 cookie 都会绑定单一的域名，无法在别的域名下获取使用

### 1.2 Session 简介

- session 是另一种记录服务器和客户端会话状态的机制
- session 是基于 cookie 实现的，session 存储在服务器端，sessionId 会被存储到客户端的cookie 中

### 1.3 区别

- JWT具有加密签名，Session和cookie没有

- JWT身份验证在本地，通过不断的传递Token不需要在服务端存储，可以对用户进行多次身份验证。Session和Cookie验证的过程会消耗大量的服务器资源。

- Session和Cookie只能用在单个节点的域或者它的子域中有效，第三个节点访问会被禁止，不可跨域。JWT支持跨域认证，能够通过多个节点进行用户认证，就是跨域认证。

## 二、为什么使用JWT

以下是使用 JWT 的一些好处：

1. 无状态：JWT 本身是无状态的，即服务器不需要在其数据库中保存会话信息。这意味着可以轻松地扩展应用程序而不必担心与会话相关的问题。这也使得在多个服务器之间平衡负载更加容易。

2. 跨平台：JWT 是基于标准化的 JSON 所构建，因此它可以与任何语言和框架无缝交互。

3. 安全性：JWT 在创建时使用密钥进行签名，因此可以确保数据没有被篡改或伪造。这可以防止中间人攻击和 CSRF 攻击等常见的 Web 安全问题。

4. 可定制性：JWT 允许您添加自定义声明作为负载中的属性，从而在应用程序中实现更多功能。

综上所述，使用 JWT 可以提高应用程序的安全性，可扩展性和跨平台兼容性，并且可以帮助开发人员解决许多常见的 Web 安全问题。

## 三、JWT的构成

JWT 由三个部分组成：

1. Header：包含令牌类型和所使用的算法
2. Payload：也称为声明，包含了需要传递的数据，如用户ID、过期时间等。
3. Signature：使用 secret key 签名后得到的哈希值，用于验证消息的完整性和真实性。

下面是一个JWT的示例：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxOTI1MXFxLmNvbSIsImFkbWluIjp0cnVlfQ.RHJtIIe8J2R5ZTh1je4IFXjCFvEJhzMAqTMppK1oDLw
```

## 3.1 Header

jwt的头部承载两部分信息：

- 声明类型，这里是jwt
- 声明加密的算法 通常直接使用 HMAC SHA256

完整的头部就像下面这样的JSON：

```
{
  'typ': 'JWT',
  'alg': 'HS256'
}
```

alg属性表示签名的算法(algorithm),默认是 HMAC SHA256(写成 HS256)typ属性表示这个令牌(token)的类型(type)。

### 3.2 Payload

JWT的第二部分是payload，也被称为claims。它是一个JSON对象，包含了关于身份验证主题（通常是用户）和其他元数据的声明信息。这些声明可能包括用户ID、角色、权限、过期时间等信息。

载荷就是存放有效信息的地方,这些有效信息包含三个部分:

- 标准中注册的声明
- 公共的声明
- 私有的声明

```
{
  "sub": "1234567890",
  "name": "abcd",
  "admin": true
}
```

### 3.3 Signature

jwt的第三部分是一个签证信息，这个签证信息由三部分组成：

- header (base64后的)
- payload (base64后的)
- secret

将这三部分用`.`连接成一个完整的字符串,构成了最终的jwt

## 四、集成JWT

引入依赖：

```
Install-Package System.IdentityModel.Tokens.Jwt
Install-Package Microsoft.AspNetCore.Authentication.JwtBearer
```

然后添加配置：

![6](https://img-blog.csdnimg.cn/22f783b05647415e8f1e27025b867e0a.png)

在这个TokenValidationParameters对象中，我们设置了三个标志分别用于验证JWT的Issuer(验证颁发者)、Audience(验证受众)和Lifetime(验证令牌的有效期)属性。因为我们希望在此示例中不验证这些属性，所以将它们全部设置为false。用于JWT签名验证在应用程序中配置。

对于JwtBearer 验证方式可以通过AddAuthentication配置多个。

其中还有：

1. `RequireExpirationTime`：是否要求令牌具有过期时间。
2. `ClockSkew`：允许的时钟偏差量，即可接受的客户端和服务器之间的时间差。
3. `RequireSignedTokens`：是否要求令牌签名。
4. `RoleClaimType`：角色声明类型。
5. `NameClaimType`：名称声明类型。

通过以上步骤的配置，我们就可以在ASP.NET Core应用程序中使用JWT身份验证了。

在项目中配置一个工具类用于封装Key：

![jwt配置类](https://img-blog.csdnimg.cn/8268f1807068453a9f2b5a6ac66038e4.png)

在生成Token方法中获取：

```csharp
var key = Encoding.UTF8.GetBytes(_jwtSettings.Value);//value不能低于32byte
```

SecurityTokenDescriptor是用于描述和生成JSON Web Tokens（JWTs）的类。JWTs是一种安全令牌，它包含有关身份验证和授权的信息，并可以在不同的应用程序之间进行共享。

生成SecurityTokenDescriptor类：

```csharp
var tokenDescriptor = new SecurityTokenDescriptor
        {
            Subject = new ClaimsIdentity(new[]
            {
                new Claim(ClaimTypes.Name, user.UserName),
                new Claim(ClaimTypes.NameIdentifier, user.Id.ToString())
            }),
            IssuedAt = DateTime.UtcNow,
            NotBefore = DateTime.UtcNow,
            Expires = DateTime.UtcNow.Add(_jwtSettings.ExpiresIn),
            SigningCredentials = new SigningCredentials(new SymmetricSecurityKey(key),
                SecurityAlgorithms.HmacSha256Signature)
        };
```

- Subject：用于指定令牌的主题，其中包含ClaimsIdentity对象，该对象代表了已认证的用户或实体的标识信息。
- IssuedAt：用于指定JWT的发行时间，通常使用UTC时间格式表示。
- NotBefore：用于指定JWT生效的最早时间，即在此之前JWT都是无效的，通常使用UTC时间格式表示。
- Expires：用于指定JWT过期的时间，即在此之后JWT都是无效的，通常使用UTC时间格式表示。
- SigningCredentials：用于指定JWT签名所需的密钥和签名算法，通常使用对称加密算法或非对称加密算法来确保JWT的安全性。

最后创建 `JwtSecurityTokenHandler` 类的实例。

```csharp
var jwtTokenHandler = new JwtSecurityTokenHandler();
```

然后，通过调用 `jwtTokenHandler` 对象的 `CreateToken()` 方法，并传入 `tokenDescriptor` 参数，创建一个 JWT 安全令牌。`tokenDescriptor` 包含关于令牌声明的信息，例如发行者、过期时间、受众以及任何用户定义的声明。

```csharp
var securityToken = jwtTokenHandler.CreateToken(tokenDescriptor);
```

使用 `jwtTokenHandler` 对象的 `WriteToken()` 方法和 `securityToken` 参数生成 JWT 的序列化字符串表示形式。该字符串可以作为身份验证令牌用于授权用户访问 Web API 或其他受保护的资源。

```csharp
var token = jwtTokenHandler.WriteToken(securityToken);
```

**配置`HttpContextAccessor`来管理请求上下文验证是否在应用程序登陆授权：**

1.在应用程序的`Startup.cs`文件中，将`HttpContextAccessor`服务添加到DI容器中

```csharp
services.AddHttpContextAccessor();
```

2.创建一个名为`CurrentUser`的新类，并在其中实现`ICurrentUser`接口

```csharp
public class CurrentUser : ICurrentUser
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public CurrentUser(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    //通过访问HttpContext属性，我们可以获取当前HTTP请求的上下文，并从中检索有关已验证用户的信息。
    public int UserId => int.Parse(_httpContextAccessor.HttpContext!.User.FindFirst(ClaimTypes.NameIdentifier)!.Value);
}
```

一般我们可以通过一个服务扩展类进行扩展以方便后续进一步对服务进行增强或者修改。所以我们在封装类中定义：![123](https://img-blog.csdnimg.cn/fba909c1efd04628868f4432c0c4efbe.png)

现在，就可以在控制器或任何需要访问当前用户ID的位置注入`ICurrentUser`服务，并直接访问其`UserId`属性。



bearer由header、Payload、Signature等核心组成，声明类型以及服务器生成一个随机字符串或数字作为token并未涉及加密算法的选择使用，使用的HMAC SHA256是一个消息认证码（MAC）算法。客户端在之后的请求中将这个token带上，以让服务器知道它的身份。
