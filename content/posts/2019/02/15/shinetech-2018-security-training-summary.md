---
title: "关注软件开发中的安全"
date: 2019-02-05T00:00:00+08:00
draft: false
type: "post"
tags:
    - "Software Security"
    - "Speech"
---

18 年 10 月底的时候，我在 Shinetech Software 内部做了一场在线的培训，主要关注的是在软件开发过程中，对于安全方面的工程实践，并不算是很深入的探讨，更多的是一些极不易察觉但又很常见的疏忽，这篇博客整理出来。

<!--more-->

## 一切漏洞都可以是注入漏洞

### 不要相信来自用户的数据

**e.g. SQL**

如果我们碰到了如下一段代码片断，用于实现将用户通过 id 从数据库中查询出来，有经验的你一定知道有什么严重的问题：

```sql
var id = ctx.query['id'];
SELECT * FROM USERS WHERE ID = id;
```

这段代码从用户的 http 请求中取出 id(查询字符串)，之后然后通过 sql 拼接生产一条合法的查询语句并执行，正常的请求可能是这样：

```
GET /users?id=66721855
[{
    "id": 66721855,
    "name": "lurongkai",
    "addr": "Tian Jin"
}]
```

那么此时执行的是`SELECT * FROM USERS WHERE ID = 66721855`，这没有问题，但是如果精心构造一个特殊的 query string 值呢？

```
GET /users?id=1%20OR%201%3D1
1%20OR%201%3D1 --(uri decode) -> 1 OR 1=1
```

这样，最后执行的 sql 就变成了`SELECT * FROM USERS WHERE ID = 1 OR 1=1`，不解释了，典型的 sql 注入，一定留意。

> 想像一下`DELETE FROM POSTS WHERE ID = id`……

解决的方法其实很简单，大部分的驱动都是支持`param`构造的，比如可以这么做：

```
SELECT * FROM USERS WHERE ID = @ID
conn.execute(sql, { ID: id });
```

> 有同学会嘲笑，说这都 2019 年了，谁还在用 sql 拼接，我们都 ORM 甚至 nosql 了。别大意，有很多你想像不到的场景仍然在大量使用 sql 拼接，这是一块神奇土壤，而且 nosql 不见得一定安全，请看下一个例子。

**e.g. Mongo**

假设我们有一个不规范的查询接口，使用了 POST 来查询 payments 数据，并在 body 中传递查询的条件：

```
POST /payments
{
    "f1": 1
}
=>
[{
    "id": "5a69b8e0817870269d5fe82c",
    "amount": "100",
    "f1": 1,
    "createdAtUtc": "2018-10-01T10:10:10.000"
}]
```

返回的是通过给定的条件过滤后的数据。一般此类的接口会限定只能查询自身的数据，所以除非烂到无法拯救，通常是不会查询到其他用户的数据的。后端 api 的实现可能会是如此：

```
var q = ctx.body;
var query = { uid: uid, …q };
db.collection['payments'].find(q);
```

uid 来自 authentication/authorization 中间件。那么最终执行的 mongo 查询会是`db.collection['payments'].find({ uid: 1 f1: 1 })`。接下来我们来构造特殊数据：

```
/POST /payments
{
    uid: {
        $ne: -1
    }
}
```

执行起来会是`db.collection['payments'].find({ uid: { $ne: -1 } })`，查到了所有用户的数据。

这其实不算是个好例子，js 的锅可能更大一些。我想在这里表达的是，mongo 的**操作符**是存在注入风险的，要留意。

**e.g. XSS**

跨站脚本注入非常常见，假设有以下接口:

```
POST /blogs
{
    "title": "my blog",
    "content": "hello, shinetech"
}
```

这个接口用来发布一篇博客，当前端开始展示该博客的时候，对应的可能是这样的代码:

```html
<html>
  <head>
    …
  </head>
  <body>
    <div id="blog-body" class="container"></div>
  </body>
  <script id="template" type="x-tmpl-mustache">
    <section>{{{ body }}}</section>
  </script>
  <script>
    $.get("/blogs/1", function (data) {
      var rendered = Mustache.render(template, data);
      $("#blog-body").html(rendered);
    });
  </script>
</html>
```

我们把逻辑剥离开，那么刚才那篇博客对应的 html 是:

```html
<html>
  <head>
    …
  </head>
  <body>
    <div id="blog-body" class="container">
      <section>hello, shinetech</section>
    </div>
  </body>
</html>
```

接下来开始构造特殊数据：

```
POST /blogs
{
    "title": "my blog",
    "content": "hello, shinetech
      <script>/* DO EVIL */</script>"
}
```

那么，渲染后的 html 是:

```html
<html>
  <head>
    …
  </head>
  <body>
    <div id="blog-body" class="container">
      <section>
        hello, shinetech
        <script>
          /* DO EVIL */
        </script>
      </section>
    </div>
  </body>
</html>
```

解决的办法也是比较简单，渲染的时候不要使用`{% raw %}{{{}}}{% endraw %}`，而是使用`{% raw %}{{}}{% endraw %}`就好了。

> 这背后的逻辑其实是这样：mustache 不是个例，而是所有的前端页面的渲染都不应该相信用户的输入，都必需要做 encoding 或者危险标签剔除(这个很有难度)，如果嫌麻烦还可以使用 markdown，但不应该直接将用户的数据渲染出来。

最终处理后的 html 会变成:

```html
<html>
  <head>
    …
  </head>
  <body>
    <div id="blog-body" class="container">
      <section>
        hello, shinetech %3Cscript%3E%2F%2A+DO+EVIL+%2A%2F%3C%2Fscript%3E
      </section>
    </div>
  </body>
</html>
```

**e.g. Bounds Checking**

边界检查简直是重灾区，不少祸事都是因为没有做严格的边界检查造成的。

假设有一个用于转账的接口：

```
POST /transfer
{
    "trans": [{
        "toUser": "5a69b8f865bdcc26a2ca32ee", // user a
        "amount": 100
    }, {
        "toUser": "5a69b8f865bdcc26a2ca32ef", // user b
        "amount": 200
    }]
}
```

后端的实现是这样的（假设 js 是一门没有溢出检查的语言）：

```js
async function processTransfer(model) {
  const balance = getBalance();
  const total = _.sumBy(model.trans, (t) => t.amount);
  //假定当前用户的余额是2000
  if (total > balance) {
    throw new Error("insufficient balance ");
  }
  // maybe in transaction
  await withdraw(ctx.user.id, total);
  for (let t of model.trans) {
    await deposit(t.toUser, t.amount);
  }
}
```

如果在正常情况下，执行的结果应该是：

```
Before:
Self:                        = 2000
User a:                      = 500
User b:                      = 500

After:
Self: 2000 - ((100) + (200)) = 1700
User a: 500 + (100)          = 600
User b: 500 + (200)          = 700
```

接下来，开始构造特殊数据：

```
POST /transfer
{
    "trans": [{
        "toUser": "5a69b8f865bdcc26a2ca32ee",
        "amount": -100
    }, {
        "toUser": "5a69b8f865bdcc26a2ca32ef",
        "amount": 200
    }]
}
```

于是，结果变为:

```
Before:
Self:                         = 2000
User a:                       = 500
User b:                       = 500

After:
Self: 2000 - ((-100) + (200)) = 1900
User a: 500 + (—100)          = 400
User b: 500 + (200)           = 700
```

再比如：

```
POST /transfer
{
    "trans": [{
        "toUser": "5a69b8f865bdcc26a2ca32ee",
        "amount": 2147483647
    }, {
        "toUser": "5a69b8f865bdcc26a2ca32ef",
        "amount": 1
    }]
}
```

结果会变成：

```
Before:
Self:                       = 2000
User a:                     = 500
User b:                     = 500

After:
Self: 2000 - ((2147483647) + (1)) = -2147481648
User a: 500 + (2147483647)        = -2147483149
User b: 500 + (1)                 = 501
```

第一段数据显示，业务逻辑没有针对负值进行处理，第二段说明没有处理溢出的情况。

为什么会溢出？

32 位整型的最大值为`2^31-1 2147483647`，最小值为`-2^31 -2147483648`，我们举例说明`Int32.MaxValue + 1`的计算过程:

```
原码:
    0_1111111111111111111111111111111
+   0_0000000000000000000000000000001
=   1_0000000000000000000000000000000
```

之后对结果（补码）转换为原码：

```
补 -> 原(后31位取反加1)
=   1_10000000000000000000000000000000
=   - 2^31 = -2147483648
```

再举个`2147483647 + (-2147483648) = -1`的例子：

```
运算时:
    0_1111111111111111111111111111111
+   1_0000000000000000000000000000000（内存中，负数都是补码表示）
=   1_1111111111111111111111111111111

补 -> 原:
    1_0000000000000000000000000000001
    - 1
```

所幸，现在的编程语言大多对溢出有了很好的控制，例如会抛出异常，但是针对边界检查上仍然需要留意。

### 不要相信外部(第三方)的数据

**e.g. 微信支付**

一般的流程是：

1. web|app: GET /wx/params 获取支付参数
2. web: (jsapi) 跳转到微信支付页面, app: (唤起微信)
3. 用户在页面或者 app 完成支付并跳回 web|app
4. 服务端等待微信服务器的回调，并完成对应业务
5. web|app 拉取最新数据

通常的实现为:

```js
// 微信支付回调接口
router.post("/wx/cb", async function (ctx) {
  const rc = ctx.request.body.return_code || "" === "SUCCESS";
  const rm = ctx.request.body.return_msg || "" === "OK";
  if (!rc || !rm) {
    // 支付失败的业务逻辑，并回复微信服务器
    ctx.body = xml(`
<xml>
  <return_code><![CDATA[SUCCESS]]></return_code>
  <return_msg><![CDATA[OK]]></return_msg>
</xml>100
`);
  }

  // 更新数据状态 & 更新订单或者其它业务
});
```

那么我们看看，发现支付的回调接口没有验证来源，其实是可以伪造支持通知请求：

```bash
curl \
  -H "Content-Type:application/json" \
  -X POST \
  --data '{ \
    "return_code": "SUCCESS", \
    "return_msg": "OK", \
    "appid: "", \                    // 可从params里取得
    "mch_id: "", \                   // 可从params里取得
    "nonce_str: "", \                // 可从params里取得
    "sign: "", \                     // 伪造
    "openid": "", \                  // 可从params里取得
    "trade_type: "", \               // JSAPI、NATIVE、APP
    "bank_type: "CMB_CREDIT", \      // 伪造
    "total_fee: 100, \               // 可从params里取得
    "transaction_id: "wx2018100110101082741", \ // 伪造
    "out_trade_no: "", \             // 可从params里取得
    "time_end: "20181001101010" \
  }' \
  http://demo.api.com/wx/cb
```

对于回调类，尤其是第三方的数据来源，一定要做身份或合法性验证。

### 也不要过于相信自己的数据

**e.g. 运营商劫持**

很多只使用`http`协议的 app 会经常发现页面上有很多第三方的广告或`widget`这是由于运营商的动态注入造成的。

<img src="/photos/2019-01-15-shinetech-2018-security-training-summary/op_hijack1.png" width="300px">
<img src="/photos/2019-01-15-shinetech-2018-security-training-summary/op_hijack2.png" width="300px">

这些 http 流量在运营商看来就是透明的，所以为了防微杜渐，干脆全走`https`更加简单。

**e.g. 双向验证**

启用`https`后事情还没有结束，有可能还会涉及到中间人攻击问题，尤其是 app 的后端 api，传统的过程是这样:

```
Browser <------- HTTPS -------> Server
```

被中间人攻击后就变成：

```
Browser <-------HTTPS -------> (Attacker)
                                    |
                                    | // 证书替换?
                                    |
Server <------- HTTPS -------> (Attacker)
```

解决的办法倒也是简单，直接将公钥打包在 app 中，通讯时强制检查公私钥就行，会安全许多。

## 一切(非业务)缺陷都是缺少检查

### 前端检查

前端页面的检查是很有必要的，除了能改善用户体验，还能部分避免后端脆弱所造成的严重问题。但是不能只依靠前端的检查，如果前端/后端的检查有个比重的话，我可能会给出`1:9`这么毫不夸张的比例：

- input required?
- input type?
- input format? email, phone, number precision/max/min
- password complexity?
- input dependencies?
- date format?
- timezone convert?
- ...

如果忽略这些

### 后端验证

当数据是从用户端产生并提交到服务端时，需要做很多的合法性检查，因为你无法盲目的去相信用户数据：

- illegal fields cleaning
- format checking
- required/dependencies checking
- range/bounds checking
- signature/certification verifying
- anti-forgery token checking(防止重放)
- business logic precondition checking
- ...

能做到这些，就能避免很大一部分的逻辑缺陷和试探。

**e.g. asp.net core mvc**

一个良好实现后端检查的例子（使用 C#语言）

```csharp
public class BookInputModel {
    [Required]
    public string Title { get; private set; }
    [Required]
    [RegularExpression(@"^\d{3}-\d-\d{3,4}-\d{4,5}-\d$")]
    public string Isbn { get; private set; }
    [Required]
    [Range(0, 1000)]
    public int Price { get; private set; }
}

[Route("api/books")]
[ApiController]
[AutoValidateAntiforgeryToken]
public class BooksController : ControllerBase
{
    [HttpPost]
    [Route("")]
    public async Task<IActionResult> Post([FromBody]BookInputModel model) {
        if (!ModelState.IsValid) {
            return BadRequest("book data invalid");
        }

        if (await _bookRepository.HasByIsbn(model.Isbn)) {
            return BadRequest("the specified book already exists");
        }

        var serviceDto = Mapper.Map<Book>(model); // 数据清洗
        var book = await _bookService.Create(serviceDto);
        var bookDisplayModel = Mapper.Map<BookDisplayModel>(book); // 数据清洗
        return CreatedAtAction("Get", bookDisplayModel);
    }

    [HttpGet]
    [Route("{id}")]
    public async Task<IActionResult> Get([FromRoute]string id) {
        var book = await _bookRepository.Find(id);
        if (null == book) {
            return NotFound();
        }

        return Ok(Mapper.Map<BookDisplayModel>(book));
    }
}
```

### 复杂的业务状态检查利器:状态机

如果我来推荐的话，对于复杂一些的业务，推荐使用状态机建模，避免自顶向下一条龙式的实现，极易出错而且还不易排查和文档化。

一个假想的例子是这样的：

```js
router.put("/some/business", async function (ctx) {
  const input = ctx.request.body;
  if (!isValid(input)) {
    throw new Error("input invalid");
  }

  const stateInput = parseRaw(input);
  const state = await loadState(stateInput);
  const expected = await decideNext(input.action);

  if (!(await state.canProcess(expected, input))) {
    throw new Error("business checking failed, forbidden");
  }

  await state.goNext(expected, input);
  await saveState(state);

  const ss = state.snapshot;
  const mapper = new Mapper();
  const res = mapper.mapTo({ ns: "some/business", data: ss });
  ctx.body = res;
});
```

当然，并不存在这样一种现成的框架，但是如果有精力，我可能会去做这么一个基础模块出来，但是背后的理念是通用的。

## 人的因素

### 弱口令

有没有见过下面这些眼熟的密码？

```
12345678
1234567890
111111
aaaaaa
1qaz2wsx#EDC
admin
hello2018
[name]112233
[phone]
```

建议还是花一些时间，对内部使用的密码例如数据库连接密码等集中做一次审计，避免因弱口令造成的事故。如果有能力的话，同时建议定期更换密码，并限制到极少数人知道或者通过安全的硬件保存。

### 未验证的第三库引用

一些关于引用第三库的小 tips：

- Open Source 库可能更安全一些
- 下载二进制包时，注意和官方的 gpg 签名做对比(有良心的都提供)
- 去官方下载，不要去 xx 网盘，宁肯使用有节操的镜像节点
- 在使用 npm, maven, nuget 等 registry 时，如果下载的是小众包，那么仍然需要慎重，你看到的代码不代表是包中使用的代码，不放心请手动 build
- 使用互操作时，请更加留意底层的原生库，一个不留神，反手就是一个缓冲区溢出，尽量避免 Interop
- docker 镜像请使用官方出品，小众镜像请手动 docker build(这事儿爆过很严重的事件: ref-> docker123321)

### credentials 及源代码管理

一些关于凭证和 SCM 的小 tips：

- 不要以明文存密码，使用 hash，必要时还可以加 salt
- CI/CD 上使用的凭证要注意隐藏，否则会在 build 日志中看到，例如使用 Jenkins 的凭证管理
- 调用第三方 api 的凭证不要直接放在源代码中，环境变量是一种选项，或者更好的配置管理方式
- 目前 github 会帮你扫描是否有敏感信息签入代码库，但是仍然需要留意，避免出现第二个"华住"
- 不要轻易把私有仓库变成公有仓库，你并不是很清楚提交历史里有什么敏感信息，有必要的话清除历史并检查后新开仓库
- 保护好企业的自动化工具，提升工具的安全性，例如保护好 Jenkins，保护好内部 npm、maven、nuget

### 一定要做环境隔离!

一些其它的 tips：

- 生产、测试、开发，不同环境应该使用的所有凭证都不相同
- JWT 签发 token 一定留意 Audience，子站 A 签发的 token 不见得是给子站 B 用的
- 不同的环境使用不同的 JWT Signing Key，这是信仰
- 除非不得已，不同的环境间应该完全阻止互通，防止跳板攻击
- 生产应该只有极少数人可以操作，尽量避免开发人员操作
- 不要在生产环境上调试错误，除非根本没在代码中做日志
- 不要在生产上安装各种无用的东西，比如安个 360 浏览器…再比如安个不知名的根证书，脆弱就是一瞬间
- 必要时，连 source-map 都不要暴露在生产上，dbg 符号文件更不能泄露，否则和泄露代码一样

## 在实践中

一些在实践中的小 tips：

- 慎重选择加密方式，必要的话请使用高强度的非对称加密，并妥善保存私钥
  代码中的写的日志一定要定期的梳理，不要把敏感信息(例如用户密码)写到日志中(github 自己出过这问题)
- 定期升级依赖库，虽然有可能会有 bug，但是和安全比起来，仍然值得投入
  类似 HeartBleed 的漏洞不是普通开发者能避免的，我们能做的就是第一时间升级，然后做好自己的安全
- 开发时，时刻进行人格分裂，站在攻击者的角度，思考一下可能会如何攻击

## 最后

需要强调两点：

- 安全投资是很昂贵的，我们能做的只能是尽量从开发层面降低伤害
- 开发环节、运维环节、基础设施，任何一层都有可能会出现问题

强烈建议！使用容器或虚拟化技术。

> 安全遵循木桶理论，缺一块儿都会降低整体可用性