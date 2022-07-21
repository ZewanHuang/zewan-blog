---
layout:     post
title:      Vue 和 Django 实现 Token 身份验证
subtitle:   "Token Authentication implemented with Vue and Django"
date:       2022-05-11 10:16:00
author:     "Zewan"
catalog:    true
mathjax:    true
header-style: text
tags:
    - Web
    - Vue
    - Django
---

使用 Django 编写的 B/S 应用通常会使用 Cookie + Session 的方式来做身份验证，用户登录信息存储在后台数据库中，前端 Cookie 也会存储少量用于身份核验的数据，由后台直接写入。但是在开发调试阶段，使用 Postman 等请求工具请求登录时，可能会缺失前端本应存储的数据，而导致登录信息核验一直不成功。在本地联调前后端时可能也会有问题。

本篇介绍基于 Token 的身份验证机制，并使用 Vue 和 Django 实现。

## 基于 Token 的验证流程

与 Session 不同的是，Token 机制不会将用户登录信息存储在后台数据库中，而是生成含有身份信息的 Token 字符串存储在前端中。在前端请求需要验证的后台 API 时，后端将优先拦截并核验身份信息。

基于 Token 的验证流程如下：

1. 客户端使用用户名和密码请求登录
2. 服务器收到请求后，验证用户名和密码
3. 验证成功后，服务端根据用户信息签发一个 Token，返回给客户端
4. 客户端存储 Token
5. 客户端每次向服务器发送其它请求时，都要携带 Token
6. 服务器收到请求，若请求的 API 需要验证身份，则先验证 Token，成功后再返回数据

<img alt="token.svg" src="/img/in-post/post-web/token.svg" style="width: 70%">

## Token 的组成

构造 Token 的方法较多，只要客户端和服务端约定好了生成和验证的格式，则有很多自定义的方法。当然，也有一些标准的写法，例如 JWT，读作 /jot/，表示 JSON Web Tokens。

JWT 标准的 Token 有三个部分：

- header
- payload
- signature

三个部分使用 `.` 分隔开，且使用 Base64 编码，示例如下：

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJuaW5naGFvLm5ldCIsImV4cCI6IjE0Mzg5NTU0NDUiLCJuYW1lIjoid2FuZ2hhbyIsImFkbWluIjp0cnVlfQ.SwyHTEx_RQppr97g4J5lKXtabJecpejuef8AqKYMAJc
```

### Header

Header 主要蕴含两部分内容的信息，分别是 Token 的类型和加密使用的方法。

初始数据对象示例如下：

```json
{
    "typ": "JWT",
    "alg": "HS256"
}
```

上述数据对象在经过算法加密、Base64 编码后，变为 Token 的第一部分：

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```

### Payload

Payload 为 Token 的具体内容，下面是可选的标准字段，也可以自定义添加需要的内容。

- iss: Issuer, 发行者
- sub: Subject, 主题
- aud: Audience, 观众
- exp: Expiration time, 过期时间, 可为时间戳格式
- nbf: Not before
- iat: Issued at, 发行时间, 可为时间戳格式
- jti: JWT ID

同样的，该部分初始数据对象经过算法加密、Base64 编码后，变为 Token 的第二部分。

### Signature

Signature 为 Token 的签名部分，相当于是前两部分的签名，用于防止其他人篡改 Token 中的信息。在处理时，可以将生成的 Token 前两段内容，使用 MD5 等签名算法进行处理，将结果作为本部分内容。

## 加密算法

从上面对 Token 组成部分的介绍中，可以了解到，在规定 Token 需蕴含的数据信息后，需要经过一定的算法加密、Base64 编码后成为 Token 的第一、二部分。因此，在生成 Token 时，要解决使用什么加密算法。

此处的加密算法一定要是可逆的、可解密的，因为我们不仅要生成 Token，还要能从 Token 中解析出我们生成时存储的数据，以验证用户信息和 Token 的有效期。因此，这里不能采用 MD5、SHA1 这样的哈希算法，因为它们无法解密，只能用于生成签名。

在 Django 中内置了加密模块 `django.core.signing`，我们调用其中的 `dumps` 和 `loads` 函数实现加密和解密。

示例：

```python
from django.core import signing
data = {
    "username": "Zewan"
}
value = signing.dumps(data) # encrypt
raw = signing.loads(value)  # decrypt
print(value, src)
```

## Django 生成和验证 Token

上面我们已经了解了 Token 机制的流程和采取的加密算法，接下来介绍 Django 中如何编写代码以实现 Token 机制。

我规定 Token 的 Header 部分为 `{"typ": "JWP", "alg": "default"}`，Payload 部分含有用户名 `username` 和过期时间 `exp`，Signature 我使用 MD5 算法生成签名。在登录成功后，后端返回给前端 username 和 Token，由前端存储起来；当前端发送需要验证身份信息的请求时，将 username 和 Token 加入请求头中，后端从请求头获取这两部分，从 Token 中解析得到用户名和过期时间，核验请求头中的 username 是否正确及 Token 是否有效。

### 处理 Token

我在 `utils/token.py` 文件中实现 Token 的生成和解析数据的功能：

```python
import time
from django.core import signing
import hashlib

HEADER = {'typ': 'JWP', 'alg': 'default'}
KEY = "Zewan"
SALT = "blog.zewan.cc"

def encrypt(obj):
    """加密：signing 加密 and Base64 编码"""
    value = signing.dumps(obj, key=KEY, salt=SALT)
    value = signing.b64_encode(value.encode()).decode()
    return value

def decrypt(src):
    """解密：Base64 解码 and signing 解密"""
    src = signing.b64_decode(src.encode()).decode()
    raw = signing.loads(src, key=KEY, salt=SALT)
    return raw

def create_token(username):
    """生成token信息"""
    # 1. 加密头信息
    header = encrypt(HEADER)
    # 2. 构造Payload(有效期14天)
    payload = {"username": username, "iat": time.time(), 
               "exp": time.time()+1209600.0}
    payload = encrypt(payload)
    # 3. MD5 生成签名
    md5 = hashlib.md5()
    md5.update(("%s.%s" % (header, payload)).encode())
    signature = md5.hexdigest()
    token = "%s.%s.%s" % (header, payload, signature)
    return token

def get_payload(token):
    """解析 token 获取 payload 数据"""
    payload = str(token).split('.')[1]
    payload = decrypt(payload)
    return payload

def get_username(token):
    """解析 token 获取 username"""
    payload = get_payload(token)
    return payload['username']

def get_exp_time(token):
    """解析 token 获取过期时间"""
    payload = get_payload(token)
    return payload['exp']

def check_token(username, token):
    """验证 token：检查 username 和 token 是否一致且未过期"""
    return get_username(token) == username and get_exp_time(token) > time.time()
```

### 登录成功批发 Token

在登录请求处理函数中，验证用户名和密码成功后，调用 `utils/token.py` 文件中的 `create_token` 函数生成 token，并将 username 和 token 返回前端。代码较为简单，此处不多展示。

### 中间件拦截验证 (Middleware)

创建一个中间件(Middleware)，在前端请求需要身份核验的后端路由时，由该中间件核验其 username 和 token，验证成功后再放行，进入业务处理的 API 中。

我的项目名称为 `backend_demo`，在自动生成的 `backend_demo` 包中，我创建 `middleware.py` 文件，构建中间件。该文件内容如下：

> 提示：前端向请求头添加 xxx 信息，一般会自动转变为 HTTP_XXX (全部大写)

```python
from utils.token import check_token
from django.http import JsonResponse

try:
    from django.utils.deprecation import MiddlewareMixin  # Django 1.10.x
except ImportError:
    MiddlewareMixin = object

# 白名单，表示请求里面的路由时不验证登录信息
API_WHITELIST = ["/api/user/login", "/api/user/register"]

class AuthorizeMiddleware(MiddlewareMixin):
    def process_request(self, request):
        if request.path not in API_WHITELIST:
            # 从请求头中获取 username 和 token
            username = request.META.get('HTTP_USERNAME')
            token = request.META.get('HTTP_AUTHORIZATION')
            if username is None or token is None:
                return JsonResponse({'errno': 100001, 'msg': "未查询到登录信息"})
            else:
                # 调用 check_token 函数验证
                if check_token(username, token):
                    pass
                else:
                    return JsonResponse({'errno': 100002, 
                                         'msg': "登录信息错误或已过期"})
```

实现中间件后，将其添加进项目中，在 `settings.py` 文件的 `MIDDLEWARE` 中添加建立的中间件：

```python
MIDDLEWARE = [
    'backend_demo.middleware.AuthorizeMiddleware',
    # ...
]
```

## Vue 存储和携带 Token

### 登录成功存储 Token

登录成功后，前端获取并存储后端返回的 username 和 token。前端实现存储的方式有很多，我这里使用简单的方法，将其存储在 localStorage 中。

```js
// 此处用 username 和 authorization 表示，放到项目中要依据情况修改该变量标识
localStorage.setItem("username", username);
localStorage.setItem("authorization", authorization);
```

### 请求头携带用户名和 Token

接下来实现请求头携带用户名和 Token 信息。

在 Vue.js 实现的项目中，我一般是用 Axios 向后端发送请求。在 Axios 中，不需要在每一处请求的代码中添加请求头代码，只需要在 `main.js` 中配置 Axios 的默认请求器，即可使所有的请求中 headers 都携带用户名和 token。

`main.js` 中核心代码如下：

> 提示：这里填入 headers 中虽然是 `username` 和 `authorization`，但会被自动转化为 `HTTP_USERNAME` 和 `HTTP_AUTHORIZATION`

```js
import axios from 'axios';

// add username and token into headers
axios.interceptors.request.use(
    config => {
        var username = localStorage.getItem('username');
        var authorization = localStorage.getItem('authorization');
        // 若 localStorage 中含有这两个字段，则添加入请求头
        if (username & authorization) {
            config.headers.authorization = authorization;
            config.headers.username = username;
        }
        return config;
    },
    error => {
        return Promise.reject(error);
    }
);
```

这样，就在 Vue.js 和 Django 编写的前后端项目中，实现了基于 Token 的身份验证机制。其他前后端框架的 Token 实现原理与本文一致，但是代码需要根据所用框架进行合理修改。
