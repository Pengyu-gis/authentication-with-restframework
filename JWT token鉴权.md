# JWT鉴权模块

	JWT(JSON WEB TOKEN)：JSON网络令牌，JWT是一个轻便的安全跨平台传输格式，定义了一个紧凑的自包含的方式在不同实体之间安全传输信息（JSON格式）。
JWT是在Web环境下两个实体之间传输数据的一项标准。实际上传输的就是一个字符串。
广义上讲JWT是一个标准的名称；狭义上JWT指的就是用来传递的那个token字符串。


## JWT的作用

由于http协议是无状态的，所以可以认为客户端和服务端的所有交互都是新的请求，这就意味着当我们通过账号密码验证用户时，当下一个request请求时它就不会携带刚刚的资料，于是程序只能再次重新识别。
****JWT就是实现了以JSON的格式，在客户端和服务端安全的传输供认证使用的信息***。*





## JWT的结构

由三部分组成，中间用 . 连接

-   头部(header)
-   载荷(payload)
-   签证(signature)

> JWT的官网：[JSON Web Tokens - jwt.io](https://jwt.io/)


## 基于JWT的token身份认证方案

 ****JWT的token身份认证的流程图*

```mermaid
sequenceDiagram
Browser ->> Server: 1. 发送带有用户名密码的post请求
Server ->> Server:2.验证通过后生成JWT,<br/>并将用户信息放入JWT
Server ->>Browser:3. 将JWT存入cookie
Browser ->>Server: 4. 发送带有JWT cookie<br/>的请求
Server ->>Server: 5. 验证签名后从JWT<br>中获取用户信息

Server ->>Browser: 6. 返回响应信息

```
>**服务端验证后，将部分的用户信息存放到JWT中，也就是存在token的字符串中，比如用户的email和用户的姓名等。在鉴权的流程当中，是直接从JWT中直接获取用户信息，这样减少了对Redis缓存组件的依赖，也减少了硬件资源的投入。**

  
  
  
### 补充：JWT原理

JWT是Auth0提出的通过对JSON进行加密签名来实现授权验证的方案，编码之后的JWT看起来是这样的一串字符：

	eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ  

由  `.`  分为三段，通过解码可以得到：

#### 1. 头部（Header）

// 包括类别（typ）、加密算法（alg）；
{
  "alg": "HS256",
  "typ": "JWT"
}

jwt的头部包含两部分信息：

-   声明类型，这里是jwt
    
-   声明加密的算法 通常直接使用 HMAC SHA256
    

然后将头部进行base64加密（该加密是可以对称解密的)，构成了第一部分。

	eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9

#### 2. 载荷（payload）

载荷就是存放有效信息的地方。这些有效信息包含三个部分：

-   标准中注册声明
    
-   公共的声名
    
-   私有的声明
    

`公共的声明 ：`  
公共的声明可以添加任何的信息，一般添加用户的相关信息或其他业务需要的必要信息.但不建议添加敏感信息，因为该部分在客户端可解密。

`私有的声明 ：`  
私有声明是提供者和消费者所共同定义的声明，一般不建议存放敏感信息，因为base64是对称解密的，意味着该部分信息可以归类为明文信息。

下面是一个例子：

// 包括需要传递的用户信息；
```	
{ "iss": "Online JWT Builder", 
  "iat": 1416797419, 
  "exp": 1448333419, 
  "aud": "www.gusibi.com", 
  "sub": "uid", 
  "nickname": "goodspeed", 
  "username": "goodspeed", 
  "scopes": [ "admin", "user" ] 
  }
```

-   iss: 该JWT的签发者，是否使用是可选的；
    
-   sub: 该JWT所面向的用户，是否使用是可选的；
    
-   aud: 接收该JWT的一方，是否使用是可选的；
    
-   exp(expires): 什么时候过期，这里是一个Unix时间戳，是否使用是可选的；
    
-   iat(issued at): 在什么时候签发的(UNIX时间)，是否使用是可选的；
    

其他还有：

-   nbf (Not Before)：如果当前时间在nbf里的时间之前，则Token不被接受；一般都会留一些余地，比如几分钟；，是否使用是可选的；
    
-   jti: jwt的唯一身份标识，主要用来作为一次性token，从而回避重放攻击。
    

将上面的JSON对象进行`base64编码`可以得到下面的字符串。这个字符串我们将它称作JWT的Payload（载荷）。

	eyJpc3MiOiJPbmxpbmUgSldUIEJ1aWxkZXIiLCJpYXQiOjE0MTY3OTc0MTksImV4cCI6MTQ0ODMzMzQxOSwiYXVkIjoid3d3Lmd1c2liaS5jb20iLCJzdWIiOiIwMTIzNDU2Nzg5Iiwibmlja25hbWUiOiJnb29kc3BlZWQiLCJ1c2VybmFtZSI6Imdvb2RzcGVlZCIsInNjb3BlcyI6WyJhZG1pbiIsInVzZXIiXX0

> `信息会暴露`：由于这里用的是可逆的base64 编码，所以第二部分的数据实际上是明文的。我们应该避免在这里存放不能公开的隐私信息。

#### 3. 签名（signature）

// 根据alg算法与私有秘钥进行加密得到的签名字串；
// 这一段是最重要的敏感信息，只能在服务端解密；
	
	HMACSHA256(  
    base64UrlEncode(header) + "." +
    base64UrlEncode(payload),
    SECREATE_KEY
	)

jwt的第三部分是一个签证信息，这个签证信息由三部分组成：

1.   header (base64后的)
    
2. payload (base64后的)
    
3.  secret
    

将上面的两个编码后的字符串都用句号.连接在一起（头部在前），就形成了:

	eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJKb2huIFd1IEpXVCIsImlhdCI6MTQ0MTU5MzUwMiwiZXhwIjoxNDQxNTk0NzIyLCJhdWQiOiJ3d3cuZXhhbXBsZS5jb20iLCJzdWIiOiJqcm9ja2V0QGV4YW1wbGUuY29tIiwiZnJvbV91c2VyIjoiQiIsInRhcmdldF91c2VyIjoiQSJ9

最后，我们将上面拼接完的字符串用HS256算法进行加密。在加密的时候，我们还需要提供一个密钥（secret）。如果我们用  `secret`  作为密钥的话，那么就可以得到我们加密后的内容:

	pq5IDv-yaktw6XEa5GEv07SzS9ehe6AcVSdTj0Ini4o

将这三部分用.连接成一个完整的字符串,构成了最终的jwt:

	eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJPbmxpbmUgSldUIEJ1aWxkZXIiLCJpYXQiOjE0MTY3OTc0MTksImV4cCI6MTQ0ODMzMzQxOSwiYXVkIjoid3d3Lmd1c2liaS5jb20iLCJzdWIiOiIwMTIzNDU2Nzg5Iiwibmlja25hbWUiOiJnb29kc3BlZWQiLCJ1c2VybmFtZSI6Imdvb2RzcGVlZCIsInNjb3BlcyI6WyJhZG1pbiIsInVzZXIiXX0.pq5IDv-yaktw6XEa5GEv07SzS9ehe6AcVSdTj0Ini4o

> `签名的目的`：签名实际上是对头部以及载荷内容进行签名。所以，如果有人对头部以及载荷的内容解码之后进行修改，再进行编码的话，那么新的头部和载荷的签名和之前的签名就将是不一样的。而且，如果不知道服务器加密的时候用的密钥的话，得出来的签名也一定会是不一样的。  
> 这样就能保证token不会被篡改。

token 生成好之后，接下来就可以用token来和服务器进行通讯了。
