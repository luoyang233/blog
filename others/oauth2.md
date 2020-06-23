# 什么是OAuth2.0

> 网上一大堆有关OAuth2.0的讲解，对于一个初入了解的人，看得是晕头转向，还好有阮一峰大佬的讲解，看完顿时茅舍顿开，其中结合了阮一峰的文章和个人的见解
>
> 阮一峰大佬是以小区大门为例进行讲解，但我觉得不是足够的贴切，这里我换成了另一种形式

## 从¥说起

### 问题来源

我在**银行**存了很多**钱**

我的儿子没钱的时候，会拿着我的卡去银行**取钱**

但是我想了一想，如果我直接把密码告诉他，万一他把我的钱全部取走了怎么办

虽然很麻烦，但是如果我直接把密码告诉他，万一他把我的钱全部取走了怎么办

所以我得给银行商量一下，在不让他知道我的密码的情况下，让我儿子有取我**部分钱**的权力

### 授权机制

于是，银行给我设计了一套授权机制

- 我儿子第一次去银行的时候，银行会给我**打电话确认**，并告知我他会**取多少钱**
- 得到我的确认之后，银行就知道了他真的是我儿子，所以给他一个**“真·儿子证明”**
- 以后他就凭借**“真·儿子证明”**去银行，银行会给他一个**短期的临时密码**
- 最后他凭借证明和临时密码向银行取钱即可

## 互联网场景

- 银行——某个平台应用（例如微信）
- 钱——我在平台上的数据（例如微信中的个人信息）
- 儿子——第三方应用（某个网站，需要获取我的微信头像和姓名）

很多时候我们会遇到这么一个场景

首先登录某个第三方应用的网站（儿子），发现它可以通过微信授权登录

然后我们点进去之后，会要求输入密码或者通过微信扫码登录（打电话确认）

登录完成后，通常会列出一些授权选项，某些可以自己勾选（取多少钱）

确认授权

真儿子证明和短期密码会在后面的介绍中提到

**简单说，OAuth 就是一种授权机制。数据的所有者告诉系统，同意授权第三方应用进入系统，获取这些数据。系统从而产生一个短期的进入令牌（token，也就是前面说的短期密码），用来代替密码，供第三方应用使用。**

## 令牌与密码

令牌与密码的作用是一样的，都可以进入系统，但是有三点差异

- 令牌是短期的，到期会自动失效，用户自己无法修改。密码一般长期有效，用户不修改，就不会发生变化。
- 令牌可以被数据所有者撤销，会立即失效。以上例而言，银行可以随时取消儿子的临时密码（令牌）。密码一般不允许被他人撤销。
- 令牌有权限范围，比如只能取我的部分钱。对于网络服务来说，令牌只能取到我的部分数据，只读令牌就比读写令牌更安全。密码一般是完整权限。

上面这些设计，保证了令牌既可以让第三方应用获得权限，同时又随时可控，不会危及系统安全。这就是 OAuth 2.0 的优点。

注意，只要知道了令牌，就能进入系统。系统一般不会再次确认身份，所以**令牌必须保密，泄漏令牌与泄漏密码的后果是一样的。** 这也是为什么令牌的有效期，一般都设置得很短的原因。

## 获取令牌

> 这里只讲到了令牌的获取过程，令牌的使用方式会在后面说到

OAuth 2.0 规定了四种获得令牌的流程

- 授权码（authorization-code）
- 隐藏式（implicit）
- 密码式（password）：
- 客户端凭证（client credentials）

P.S.不管哪一种授权方式，第三方应用申请令牌之前，都必须先到系统备案，说明自己的身份，然后会拿到两个身份识别码（真儿子证明）：客户端 ID（client ID）和客户端密钥（client secret）。这是为了防止令牌被滥用，没有备案过的第三方应用，是不会拿到令牌的。

### 一：授权码

> 授权码是最常用也是最安全的授权方式，它适用于那些有后端的 Web 应用。授权码通过前端传送，令牌则是储存在后端，而且所有与资源服务器的通信都在后端完成。这样的前后端分离，可以避免令牌泄漏。上面的例子也是以授权码为例进行假设的

1.A 网站(儿子)提供一个链接，用户点击后就会跳转到 B 网站（银行），授权用户数据（钱）给 A 网站使用。下面就是 A 网站跳转 B 网站的一个示意链接。

```
https://b.com/oauth/authorize?
  response_type=code&
  client_id=CLIENT_ID&
  redirect_uri=CALLBACK_URL&
  scope=read
```

上面 URL 中，`response_type`参数表示要求返回授权码（`code`），`client_id`参数让 B 知道是谁在请求，`redirect_uri`参数是 B 接受或拒绝请求后的跳转网址，`scope`参数表示要求的授权范围（这里是只读）。

<img src="https://github.com/luoyang233/blog/blob/master/images/oauth2_1.png" alt="oauth2_1" style="zoom:50%;" />

2.用户跳转后，B 网站会要求用户登录，然后询问是否同意给予 A 网站授权(例子中的打电话确认)。用户表示同意，这时 B 网站就会跳回`redirect_uri`参数指定的网址。跳转时，会传回一个授权码(`code`)，就像下面这样。

```
https://a.com/callback?code=AUTHORIZATION_CODE
```

<img src="https://github.com/luoyang233/blog/blob/master/images/oauth2_2.png" alt="oauth2_2" style="zoom:50%;" />

3.A 网站拿到授权码以后，就可以在**后端**，向 B 网站请求令牌(临时密码)。

```
https://b.com/oauth/token?
 client_id=CLIENT_ID&
 client_secret=CLIENT_SECRET&
 grant_type=authorization_code&
 code=AUTHORIZATION_CODE&
 redirect_uri=CALLBACK_URL
```

上面 URL 中，`client_id`参数和`client_secret`参数用来让 B 确认 A 的身份（`client_secret`参数是保密的，因此只能在后端发请求），`grant_type`参数的值是`AUTHORIZATION_CODE`，表示采用的授权方式是授权码，`code`参数是上一步拿到的授权码，`redirect_uri`参数是令牌颁发后的回调网址。

<img src="https://github.com/luoyang233/blog/blob/master/images/oauth2_3.png" alt="oauth2_3" style="zoom:50%;" />

4.B 网站收到请求以后，就会颁发令牌。具体做法是向`redirect_uri`指定的网址，发送一段 JSON 数据。

```
{    
  "access_token":"ACCESS_TOKEN",
  "token_type":"bearer",
  "expires_in":2592000,
  "refresh_token":"REFRESH_TOKEN",
  "scope":"read",
  "uid":100101,
  "info":{...}
}
```

上面 JSON 数据中，`access_token`字段就是令牌，A 网站在后端拿到了。

<img src="https://github.com/luoyang233/blog/blob/master/images/oauth2_4.png" alt="oauth2_4" style="zoom:50%;" />

### 二：隐藏式

> 隐藏式主要针对纯前端，无后端的应用，省略了授权码这个步骤
>
> 这种方式是非常不安全的，一般用于安全性要求不高的场景，通过在浏览器关闭后令牌失效

1.A 网站提供一个链接，要求用户跳转到 B 网站，授权用户数据给 A 网站使用。

```
https://b.com/oauth/authorize?
  response_type=token&
  client_id=CLIENT_ID&
  redirect_uri=CALLBACK_URL&
  scope=read
```

上面 URL 中，`response_type`参数为`token`，表示要求直接返回令牌。

2.用户跳转到 B 网站，登录后同意给予 A 网站授权。这时，B 网站就会跳回`redirect_uri`参数指定的跳转网址，并且把令牌作为 URL 参数，传给 A 网站。

```
https://a.com/callback#token=ACCESS_TOKEN
```

P.S.令牌的位置是 URL 锚点（fragment），而不是查询字符串（querystring），这是因为 OAuth 2.0 允许跳转网址是 HTTP 协议，因此存在"中间人攻击"的风险，而浏览器跳转时，锚点不会发到服务器，就减少了泄漏令牌的风险。

### 三：密码式

> 举例简单来说，就是把你的银行卡账户和密码直接告诉你的儿子，也就是把你的帐号和密码直接告诉第三方，因此这种方式同样也是风险极高

1.A 网站要求用户提供 B 网站的用户名和密码。拿到以后，A 就直接向 B 请求令牌。

```
https://oauth.b.com/token?
  grant_type=password&
  username=USERNAME&
  password=PASSWORD&
  client_id=CLIENT_ID
```

上面 URL 中，`grant_type`参数是授权方式，这里的`password`表示"密码式"，`username`和`password`是 B 的用户名和密码。

2.B 网站验证身份通过后，直接给出令牌。注意，这时不需要跳转，而是把令牌放在 JSON 数据里面，作为 HTTP 回应，A 因此拿到令牌。

### 四.凭证式

> 这种方式主要针对第三方应用，而不是用户，即有可能多个用户共享同一个令牌。

1.应用在命令行向 B 发出请求。

```
https://oauth.b.com/token?
  grant_type=client_credentials&
  client_id=CLIENT_ID&
  client_secret=CLIENT_SECRET
```

上面 URL 中，`grant_type`参数等于`client_credentials`表示采用凭式，`client_id`和`client_secret`用来让 B 确认 A 的身份。

2.B 网站验证通过以后，直接返回令牌。

## 令牌的使用

> 令牌的使用理解起来，相对来说就比较简单了，在示例场景中也就是最后一步，用"临时密码"去取钱

A 网站拿到令牌以后，就可以向 B 网站的 API 请求数据了。此时，每个发到 API 的请求，都必须带有令牌。具体做法是在请求的头信息，加上一个`Authorization`字段，令牌就放在这个字段里面。

```javascript
 headers.append('Authorization', `Bearer ${ACCESS_TOKEN}`);
```

## 更新令牌

> 令牌的有效期一般很短，在令牌有效期过期后，如果再走一遍获取流程，无论对用户还是第三方应用来说都非常的不友好，所以在令牌颁布的时候一般是同时颁布两个令牌

- 普通令牌(`ACCESS_TOKEN`)：也就是上面提到的用于获取数据的令牌
- 刷新令牌(`REFRESH_TOKEN`)：用于获取新令牌的令牌，在普通令牌过期后，通过刷新令牌再次获取一个新的普通令牌

```
https://b.com/oauth/token?
  grant_type=refresh_token&
  client_id=CLIENT_ID&
  client_secret=CLIENT_SECRET&
  refresh_token=REFRESH_TOKEN
```

上面 URL 中，`grant_type`参数为`refresh_token`表示要求更新令牌，`client_id`参数和`client_secret`参数用于确认身份，`refresh_token`参数就是用于更新令牌的令牌。

P.S.在令牌过期后，应用内的请求通常会是一次无效的请求，如果再让用户重新请求一遍，会造成非常不友好的体验，所以`refresh_token`的使用通常是[利用promise无痛刷新token](https://segmentfault.com/a/1190000020210980#articleHeader11)



