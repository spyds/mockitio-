# JWT认证的使用

## 一、JWT介绍

### 1.什么是JWT？

​	JWT的英文全拼是"JSON Web Token", 也就是通过JSON形式作为Web应用中的令牌,用于在各方之间安全地将信息作为JSON对象传输。在数据传输过程中还可以完成数据加密、签名等相关处理。

### 2.JWT可以做什么？

**(1) 授权**

​	用户登录成功后，由JWT生成令牌。后续的每个请求只要Header包括该令牌，则允许用户访问该令牌允许的路由，服务和资源。

**(2) 信息交换**

​	可以利用JWT用于不同系统之间安全地传输信息。因为可以对JWT进行签名，所以您可以确保发件人是他们所说的人。此外，由于签名是使用标头和有效负载计算的，因此您还可以验证内容是否遭到篡改。

### 3.相比于传统基于Session的认证，JWT的优势是什么？

**（1）传统基于SESSION认证的工作原理**

​	HTTP协议是无状态的，为了知道发起HTTP请求是来自于哪位用户，我们会在用户登陆时保存用户登录信息于服务器的session中，同时会将session id保存于cookie返回给用户，保存于浏览器中。下一次用户携带cookie请求过来，我们根据其中的session id获取对应的session，便可从中获取用户信息。

​	**存在的问题：**

​	1.对每个用户，服务器端都要保存一份session，随着用户的增多，系统内存也会增大

​	2.用户认证被服务器保存在内存中，这意味着用户下次请求还必须要请求在这台服务器上,这样才能拿到授权的资源，这样在分布式的应用上，相应的限制了负载均衡器的能力。这也意味着限制了应用的扩展能力。（可以通过分布式session解决）

​	3.因为是基于cookie来进行用户识别的, cookie如果被截获，用户就会很容易受到跨站请求伪造的攻击。

​	4.sessionid就是一个特征值，表达的信息不够丰富，不容易扩展。

![1611595565457](C:\Users\10560\notes\jwt.assets\1611595565457.png)

   **(2) JWT的工作原理**

**JWT的结构：**

JWT由header(标头)、payload(有效载荷)、Signature(签名)三部分构成的字符串(**head.payload.singurater**) 。

**header：**

​		typ指示令牌的类型(即为JWT)，alg指示所使用的签名加密算法(HS256，HMAC，SHA256或RSA)

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**payload：**

​	有效载荷包含用户信息或其他信息

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

**Signature：**

JWT基于BASE64分别对header，payload两部分进行编码并拼接，形成字符串：header_base64.payload_base64,  之后使用header指定的签名算法，加之我们自己提供的密钥，形成token的第三部分singurater。



**JWT的认证主要有以下几个步骤：**

​		(1) 用户登录网页时，后端系统由jwt根据用户信息和密匙按照一定的编码和加密算法生成token (head.playload.singurater格式的字符串) 返回给前端

​		(2) 以后每次前端发起请求时便携带该token，后端解码该token只要符合编码规律，即可认为其通过校验。

​	**总结：**这里可以做个总结，jwt就是一个编解码加密的工具

**JWT的优势：**

​	（1）简洁：token可以作为POST参数或者在HTTP header发送，数据量小，传输速度快

​	（2）自包含：不同于session id只是一个特征值,  而jwt可以在其载体payload包含其用户或其他信息

​	（3）跨平台：以JSON加密的形式保存在客户端的，所以JWT是跨语言的，原则上任何web形式都支持。

​	（4）jwt不用在服务器端保存session，只需浏览器保存生成的token，因此jwt尤其适合分布式的web项目

![1611935559482](C:\Users\10560\notes\jwt.assets\1611935559482.png)

## 二、JWT的使用

##### **（1）引入依赖**

```xml
		<!--引入jwt-->
        <dependency>
            <groupId>com.auth0</groupId>
            <artifactId>java-jwt</artifactId>
            <version>3.4.0</version>
        </dependency>
```

##### **（2）编写工具类**

```java
public class JWTUtils {
    /**
     * 密钥
     */
    private static final String SECRET_KEY = "&(*(DSFSD^&SDFD*F&SD*";
    /**
     * 签名加密算法
     */
    private static final Algorithm ALGORITHM = Algorithm.HMAC256(SECRET_KEY);
    /**
     * token有效时间(单位：毫秒)
     */
    private static final Long TOKEN_ACTIVE_TIME = 1000L * 60L * 30L;

    /**
     * 获取Token
     */
    public static String getToken(User user) {
        // 1.获取JWT构造器
        JWTCreator.Builder builder = JWT.create();
        // 2.构建负载payload
        Date loginTime = new Date();
        if (user == null) {
            user = new User();
        }
        user.setLoginTime(loginTime);
        builder.withClaim("id", user.getId());
        builder.withClaim("name", user.getName());
        builder.withClaim("loginTime", user.getLoginTime());
        // 3.设置token过期时间
        builder.withExpiresAt(new Date(loginTime.getTime() + TOKEN_ACTIVE_TIME));
        // 4.签名并生成token
        return builder.sign(ALGORITHM);
    }

    /**
     *  解析token并获取其负载
     */
    private static User resolveToken(String token) {
        try {
            DecodedJWT decodedJWT = JWT.require(ALGORITHM).build().verify(token);
            User user = new User();
            user.setId(decodedJWT.getClaim("id").asLong());
            user.setName(decodedJWT.getClaim("name").asString());
            user.setLoginTime(decodedJWT.getClaim("loginTime").asDate());
            return user;
        } catch (AlgorithmMismatchException e) {
            throw new RuntimeException("签名算法不匹配");
        } catch (SignatureVerificationException e) {
            throw new RuntimeException("签名有误", e);
        } catch (TokenExpiredException e) {
            throw new RuntimeException("签名过期", e);
        }
    }

    public static void main(String[] args) {
        User user = new User();
        user.setId(888L);
        user.setName("name");

        String token = getToken(user);
        System.out.println("生成的Token为:" + token);

        User result = resolveToken(getToken(user));
        System.out.println(result);
    }
}
```

**输出结果为：**

```bash
生成的Token为:
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblRpbWUiOjE2MTE5OTk0ODEsIm5hbWUiOiJuYW1lIiwiaWQiOjg4OCwiZXhwIjoxNjEyMDAxMjgxfQ.QLELKjRaVbLYhNlkAZ_KU1E1e-05ujYZRRqlv9tZ4Z8

User(id=888, name=name, loginTime=Sat Jan 30 17:38:02 CST 2021)
```



##### **（3）编写拦截器**

```java
@Slf4j
public class JwtInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String token = request.getHeader("token");
        try {
            User user = JWTUtils.resolveToken(token);
            SecurityContext.holdUser(user);
        } catch (Exception e) {
            log.info("登录失败：{}", e.getMessage());
            return false;
        }
        return true;
    }
}
```

```java
@Configuration
public class InterceptorConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new JwtInterceptor()).addPathPatterns("/**").excludePathPatterns("/AppController/login", "/");
    }
}
```



##### **（4）编写登录接口**

```java
@PostMapping(LOGIN)
public String login(User user, ModelMap map) {
    String token = JWTUtils.getToken(user);
    map.addAttribute("token", token);
    return "login";
}
```

