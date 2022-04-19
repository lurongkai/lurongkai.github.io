---
title: "如何优雅的在 Golang 中进行错误处理"
date: 2019-07-26T00:00:00+08:00
draft: false
type: "post"
tags:
    - "Golang"
    - "Microservice"
---

如何优雅的在`Golang`中进行错误处理？

答案是：没有……（本文完）

---

开个玩笑，`Golang`中的错误处理方式一直是社区热烈讨论的话题，有力挺者，有抱怨者，但不论如何，自 2009 年`Golang`正式发布以来，关于错误处理就一直是现在这种状况。

随着`Golang`愈加的火爆，原本是`Java`、`Node`、`C#`等语言擅长的应用级开发领域也逐渐出现`Golang`的身影。`Golang`自身其实更加擅长做基础设施级开发，例如`docker`，例如`k8s`，再如`etcd`，它友好的内存管理和简单到粗暴的语法（25 个关键字），特别适合过去`C`和`C++`这些语言所擅长的部分场景。我们有理由相信，`Golang`下一个大的引爆点将也许会在`IoT`上，因为它天然的适合。

当一门语言火起来，就会出现各式各样的应用，于是`MVC`框架有了，音视频处理库有了，各种数据库驱动有了，甚至服务框架也出现了，游戏、`Machine Learning`都不在话下，还要啥自行车？组合一下做应用级开发妥妥的没毛病。

但是，成也这 25 个关键字，败也这 25 个关键字，究其根本原因，都是因为它背后**简单**的哲学。

做应用级开发可不是那么简单的，这涉及到很多的细节处理，例如本文将要讨论的错误处理。如果只是写一个库，那么这个话题相对比较简单，因为与`API`打交道的都是开发者，你只管开心的往外扔`error`就好了，总会有倒霉的程序员在使用你的代码时**DEBUG**到白头，最后，以最严谨的方式，小心使用你的库；可是有人出现的地方就会有幺蛾子，一个常见的误区就是将**业务错误**、**运行时错误**、**程序错误**一股脑的当成相同的`error`来处理。

<!--more-->

> 你是还没在`error`上栽跟头，当你栽了跟头时才会哭着想起来，当年为什么没好好思考和反省**错误处理**这么一个宏大的话题

那么，如何在现有的语言支持下，用一种相对优雅的方式进行错误处理呢？我们通过本文的思考和讨论，尝试予以解决。虽说主要讨论的是`Golang`，但是这背后的思考其实适合大部分语言。

## 语言级别的错误处理

`Golang`是原生支持鸭子类型（[duck typing](https://en.wikipedia.org/wiki/Duck_typing)）的，所以`error`可以理解成一个“鸭子”的定义，它是这样的：

```go
// golang
type error interface {
    Error() string
}
```

换句话讲，一切实现了`Error() string`方法的`struct`，都可以当成`error`往外扔，神不神奇？不神奇……把它看成接口也无碍，反正其它语言也长的类似，比如：

```csharp
// csharp
public class SystemException : Exception {}
public class InvalidOperationException : SystemException
```

问题来了，`Golang`是没有继承这一说的，所以如果你想把错误规划成层级结构是行不通的，而且也不是`Golang`的调调。不过定义多种错误终归是可以的：

```go
// golang
import "errors"
var (
    ErrNotAuthenticated = errors.New("not authenticated")
    ErrNotAuthorized    = errors.New("not authorized")
    ErrNoPermission     = errors.New("no permission")
)
```

鸭子类型在不使用继承的情况下变相支持了多态，所以是可以认为`error`是个接口，`error`的消费方可以不用关心背后是具体什么结构，只需要满足`error`契约就行，这就是所谓的多态。

那么到这里为止，我们有了具体的`error`，然后呢？总是得有一个地方去处理。从这里开始，`Golang`与别的语言区分开了。

本来想解释一下什么是错误(error)，什么是异常(exceptional)，但是貌似太多的语言在混搭使用这两个术语，所以我们干脆放弃解释错误和异常，而使用可恢复和不可恢复来说明。同时，我个人实名点赞`Golang`和`Rust`在这两个概念上的区分。

### 不可恢复故障

`Golang`和`Rust`都有`panic`的概念，也就是指不可恢复的故障，一般遇到`panic`时基本就不用再救了，大部分的时候都是直接以`-1`为返回值退出程序就好，**除非你觉得我行我可以我还想再试试**，那么使用`recover`手段，比如：

```go
// golang
fun horrible() {
    panic("some bad things happened")
}

func business() {
    defer func() {
        if p := recover(); p != nil {
            // give me another chance
        }
    }()
    horrible()
}
```

如果不处理的话，程序就会自动退出，并打印出错误信息以及错误堆栈。`Rust`的方式几乎一模一样，只不过有两点不同：`Rust`中对应`panic`的是`panic!`宏，和`recover`类似的功能是`std::panic::catch_unwind`；另外就是退出后默认不打印堆栈，需要的话得手动设置`RUST_BACKTRACE=1`环境变量。

这个非常好理解，比如数组越界了，内存满了，堆栈爆了，几乎碰到`panic`就很少有恢复的可能。

`panic`背后其实是一种**短路**（或者叫**快捷方式**）哲学，任何层级的流程在执行过程中，通过`panic`都可以直接让程序跳到结束或者有`recover`的地方。这与大多数据的高级语言的`Exception`不谋而合，举个例子：

```csharp
// csharp
public void DoSometingIntresting() {
    throw new InvalidOperationException("not allowed");
}
```

只要碰到`Exception`就一定会中断正常的执行顺序。稍显遗憾的是，这些语言中的`Exception`不完全能把程序打死，因为它们大多都提供了`try-catch`语言构造，让你可以在任何想处理的地方，或处理或加工，总之手法多样。打不死的原因也正是因为，一个简单的`catch (Exception ex) {}`就足够吃掉所有的故障。

这也是为什么在开头那部分里不用错误和异常的原因，因为：

> 大多数支持`try-catch-exception`机制的语言里，可恢复和不可恢复的故障都用 Exception 来表示，这加剧了开发者的心智负担，因为这需要仔细的处理 Exception 的类型。例如，C#里的**不可恢复**错误往往都有特定的继承链，比如`SystemException`，使用时需要小心处理。

更好的理解方式是，把`try-catch-exception`这种机制，主要作为处理**可恢复**故障的手段，而把少量**不可恢复**的故障，在充分思考的情况下处理或放任。换句话讲，**catch 一定尽可能的按下游方法可能出现的`Exception`类型去匹配，不要随意通吃**。

### 可恢复故障

与`panic`有所区别的**可恢复**故障，`Golang`也有约定的方式。这就是`error` 。

所谓可恢复，就是虽然无法顺利的将当前的流程执行完毕，但是不影响大局，消费方可以按自己的意愿去安排接下来的逻辑，或中断执行某个业务，或检查是否自己使用的方式有问题，或有备用的流程替换等等。

实践中经常碰到的可恢复故障有几大类：

- **前置检查失败**，大多是指参数没有按约定提供，例如参数不可空校验失败的错误，参数数值范围不正确等等，这是**调用方的 bug**
- **程序错误**，例如通过`req.(sometype)`进行类型转换，到运行时发现转不过去，这是**自身的 bug**
- **依赖服务调用错误**，比如查询数据库时发生了异常，往往都是第三方产生运行时错误，是最经常处理的错误
- **业务执行错误**，例如一个发送验证码的函数，在执行过程中发现某个用户的发送频率超过阈值，那这是一个特定业务的失败

基本所有在开发过程中碰到的错误都能归入以上 4 类。而往往需着重关注的，是后两类。前两类实属于**bug**，需要在上线前就清理完毕的。

## 可恢复故障的抛出方式

我们来做一个思考。在一门语言中，如果一个方法有可能出错，通常会通过什么途径把错误信息告诉调用者呢？换句话讲，正常的方法返回数据，不正常的方法需要有途径“带货”，把错误信息以某种方式带出去。

### 单值函数的方式

如果这门语言只支持单值函数，也就是返回值只能是一个，那么就需要有一个容器来储存正常的值和出错时需要返回的错误信息，就像：

```fsharp
// fsharp
type Result<'T> = {
    Data: 'T
    Error: string
}
```

此时，每个使用该方法的地方，只需要简单判断一下`res.Error`就能知道有没有错误发生。

像不像是很多`Restful`接口返回数据的模样？是的，完全是一个模式：

```json
{
  "data": {
    // ...
  },
  "errmsg": "",
  "errcode": "610100"
}
```

这里先忽略这个错误码，后面的内容我们会涉及到。

### 多值函数的方式

那如果语言支持多值返回（其实还是单值，大多是引入`元组（Tuple）`来处理，例如`Python`），概念上和如下的方式相同：

```csharp
// csharp
public Tuple<int, string> Multiple() {
    return Tuple.Create(0, "something wrong");
}
```

好了，该`Golang`出场了，既然我支持多值返回，那么应该不用明显的包装类型就可以做到了吧：

```go
// golang
func multiple() (int, string) {
    return 0, "something wrong"
}
```

等等，错误用`string`表示有点丑是不是，没关系，`Golang`帮你抽象出一个`error`接口来，最终就变成了`func multiple() (int, error) {}`这样子了，
和定义一个`type res struct { Data int; Err error}`相比，好像没进步太多？

### 函数式的方式

那还有没有更好的方式了呢？如果有接触过`Functional Programming`的东西，就会想到，通过`Generic` + `Discriminated Unions` + `partten matching`的方式更加优雅。

核心在`Discriminated Unions`上，也叫做`Enum`，`Union`，`Tagged Union`，`variant`，`variant record`，`choice type`，`disjoint union`，`sum type`，`coproduct`……它是一种可以存储多种（但是数量固定）类型值的结构，同一时间**只可以**使用其中的一种类型。举个例子，如下的`DUs`可以避免`null`的显式使用：

```fsharp
// fsharp
type Option<'T> =
    | None
    | Some of 'T
```

这个`Option<'T>`（也有叫`Maybe`的）要么只有`None`值，要么只有一个包含`'T`的`Some`值，于是，当函数返回一个`Option`类型的值时，消费方就可以不再写诸如`Golang`中的`if err != nil {}`了，而是使用更加高级的模式匹配完成：

```fsharp
// fsharp
let res = somemethod() // will return an Option value
match res with
    | None   -> // data is empty, like null
    | Some d -> // d is data
```

等等，这不像是在做错误处理？没关系，稍微变换一下：

```fsharp
// fsharp
type Result<'T, 'TError> =
    | Ok of 'T
    | Error of 'TError
```

现在的使用方式变成了：

```fsharp
// fsharp
let res = somemethod() // will return an Option value
match res with
    | Ok t      -> // t is normal result
    | Error err -> // err is error
```

好像还是没什么用？那是因为没有接触过`FP`中的`Warpper`类型的概念，基本上有了`Warpper`类型，就可以`bind`或者`lift`等等了。`Rust`走的就是这种路子，并有配套的函数支持。由于`Option`和`Result`如此常用，以至于很多语言核心库都内置了对应的结构，有兴趣可以参考我很早之前写过的一点[东西](http://www.ituring.com.cn/article/207638)。

那么，`Golang`为什么不使用这种方式呢？因为，第一缺乏泛型支持，`Warpper`如果没有泛型支持的话就无法泛化，会导致很多的模板代码，进而还不如直白的处理`error`；第二没有`Discriminated Unions`，多个类型无法联合起来并在同一时间只使用其中一种，也就快速区分彼此；第三没有模式匹配，也就无法更进一步的简化代码，不如还是使用`if err != nil {}`。

上面诸多方式仍然停留在`调用-返回-处理`这个流程上，顶多也就是代码简洁与否的问题。我个人是认可`Golang`的错误处理方式的，虽然会出现很多的模板代码，但是在写代码的每一步都能清晰的并强迫性的让开发者处理潜在的错误，也是一种提高质量的不错手段。

实践中使用最多的方式，是隔空传送`Exception`，虽然有很多的文章在指导大家如何去花式处理`Exception`，但是仍然值得大家留意其中的陷阱。毕竟，异常是一种中断当前执行流程的手段，并且会穿透调用栈，所以需要格外留意捕获到的异常究竟代表了什么含义，而不是一股脑的全部捕获。这一点要赞一下`Java`，`Java`中的方法签名会强制列出有可能抛出的异常类型，以供开发者快速处理可能出现的异常。

有关`Exception`设计和使用的话题，我们将来有机会再来聊。

### Golang 中将来可能的方式

在`Go 2`的草案中，我们看到了有关于`error`相关的一些提案，那就是`check/handle`函数。

我们也许在下一个大版本的`Golang`可以像下面这样处理错误：

```go
// golang
import "fmt"
func game() error {
    handle err {
        return fmt.Errorf("dependencies error: %v", err)
    }

    resource := check findResource() // return resource, error
    defer func() {
        resource.Release()
    }()

    profile := check loadProfile() // return profile, error
    defer func() {
        profile.Close()
    }

    // ...
}
```

有兴趣的同学请关注[这个提案](https://go.googlesource.com/proposal/+/master/design/go2draft-error-handling-overview.md)。题外话，还有一个`try`[提案](https://go.googlesource.com/proposal/+/master/design/32437-try-builtin.md)正式被[否了](https://github.com/golang/go/issues/32437#issuecomment-513002788)。

所以，在`Golang`中我们目前可以使用的方式，就是以`error`接口为基础，通过不同的错误类型，来向消费方提供有价值的信息。

## 可恢复故障具体该怎么抛

重点来了，说了这么多，错误终归是要扔出去的，虽然都是统一的`error`接口，但是手法却应该仔细斟酌。

### 错误应该包含的信息

错误最主要包含的，就是错误信息，是给人类阅读使用的，更确切的讲，是**给开发者阅读的**。所以`error`接口里的`Error() string`直接将这个信息返回。那为什么要返回`error`，而不是直接返回`string`呢？因为在开发过程中，我们往往需要一些额外的信息。

首先，如果只有错误的文本，我们很难定位到具体的出错地点。虽然通过在代码中搜索错误文本也是有可能找到出错地点的，但是信息有限。所以，在实践中，我们往往会将出错时的调用栈信息也附加上去。调用栈对消费方是没有意义的，从隔离和自治的角度来看，消费方唯一需要关心的就是错误文本和错误类型。调用栈对实现者自身才是是有价值的。所以，如果一个方法需要返回错误，我们一般会使用`errors.WithStack(err)`或者`errors.Wrap(err, "custom message")`的方式，把此刻的调用栈加到`error`里去，并且在某个统一地方记录日志，方便开发者快速定位问题。

举个例子：

```go
// golang
import "github.com/pkg/errors"
func FindUser(userId string) (*User, error) {
    if userId == "" {
        return fmt.Errorf("userId is required")
    }

    user, err := db.FindUserById(userId)
    if err != nil {
        return nil, errors.Wrapf(err, "query user %s failed", userId)
    }

    return user, nil
}
```

如此，在记录日志的地方通过使用`%+v`格式化占位符就可以把堆栈信息完整的记录下来。

其次，如果是业务执行时的错误，只有错误消息的话，往往是不够的，因为调用方更加关心错误背后业务上的原因，例如，提交订单接口返回了**提交订单失败**的错误，为什么失败？这个时候就需要某种机制来告诉调用者一些业务上的原因。显然，如果通过错误消息告诉的话，调用方就不得不对错误文本进行判断，这很不优雅，所以我们往往通过其它两种方式来处理。

**1. 特定错误类型**，例如：

```go
// golang
var (
    ErrInventoryInsufficient      = errors.New("product inventory insufficient")
    ErrProductSalesTerritoryLimit = errors.New("product sales torritory limit")
)

func Ordering(userId string, preOrder *PreOrder) (*model.Order, error) {
    order := &model.Order{}

    shippingAddress := preOrder.Shipping
    for _, item := range preOrder.Items {
        if findInventory(item.Product.Id) <= 0 {
            return nil, ErrInventoryInsufficient
        }

        if !isValidSalesTerritory(item.Product.Id, shippingAddress) {
            return nil, ErrProductSalesTerritoryLimit
        }

        order.AddItem(item)
    }

    // other processing
    return order, nil
}
```

这样，消费方拿到错误后，可以很简单的判断一下就能知道具体发生了什么：

```go
// golang
func UserOrderController(ctx context.Context, preOrder *PreOrder) {
    // some preparing
    user := FromContext(ctx)
    order, err := service.Ordering(user.userId, preOrder)
    if err != nil {
        switch err {
            case service.ErrInventoryInsufficient:      // handling
            case service.ErrProductSalesTerritoryLimit: // handling
        }
    }
    // ...
}
```

这也是很多组件向外部提供错误的首选方式，例如，`mongo.ErrNoDocuments`

但是遗憾的是，如果是跨边界的`RPC`调用的话（假如刚才的`Ordering`是个微服务），那么就不能采用这种方式了，因为错误**类型**是无法有效序列化的，即使序列化了也失去了类型判断的能力。所以，我们在集成有边界的服务时，往往会采用另一种方式。

**2. 错误标记**，也就是通过某种约定好的标记，用于表示某种类型的业务错误。客户端调用远程的`Restful`服务也是边界与边界间的调用，所以我们经常可以在`API`的文档中看到这样的模式：

| 返回码   | 错误码描述              | 说明              |
| ----- | ------------------ | --------------- |
| 40001 | invalid credential | 不合法的调用凭证        |
| 40002 | invalid grant_type | 不合法的 grant_type |

这里的返回码就是一种约定好的标记，也叫**业务码**。所谓跨边界调用，也可以换个说法，叫做进程间通讯，如果只在进程内通讯，那使用特定错误类型就足够了，但是一旦出了进程，就需要某种标记手段了。

`Golang`在实践中也可以采用这种方式，尤其是在边界间传递错误的时候：

```go
// golang
import (
    "fmt"
    "regexp"
)

type BusinessError struct {
    Code    string `json:"code"`
    Msg     string `json:"msg"`
}

// error interface
func (be BusinessError) Error() string {
    return fmt.Printf("[%s] %s", be.Code, be.Msg)
}

var codeReg = regexp.MustCompile("^\\d{6}$")
// factory method
func NewBusinessError(code string, msg string) *BusinessError {
    if !codeReg.MatchString(code) {
        panic("code can only contain 6 numbers")
    }

    if msg == "" {
        panic("msg is required")
    }

    return &BusinessError{ code， msg }
}

var (
    ErrInventoryInsufficient      = NewBusinessError("301001", "product inventory insufficient")
    ErrProductSalesTerritoryLimit = NewBusinessError("301002", "product sales torritory limit")
)
```

注意`NewBusinessError`内部使用的是`panic`，这背后的思考是，如果程序初始化时连错误码的定义都能出现问题，我倾向于让程序跑不起来，这样便在开发阶段就能妥善处理。

消费方拿到反序列化后的错误时，里面已经包含了标记，查询文档分别做处理就好。不管是`Restful`，还是`GRPC`、`GraphQL`，都可以使用这种模式来处理。甚至更大好处是，客户端不必判断错误文本并设法解析出用户友好的提示，服务不再提供用户提示（想想看，如果要对错误文本提供`i18n`支持的话，得多难看……），一切都交给客户端去自主选择。

### 错误信息应该暴露多少

**暴露多少错误细节，取决于对这个错误感兴趣的一方是谁。**
**暴露多少错误细节，取决于对这个错误感兴趣的一方是谁。**
**暴露多少错误细节，取决于对这个错误感兴趣的一方是谁。**

如果感兴趣一方是其他开发者，那么事情就会变的愉快很多，因为，开发者感兴趣的错误，一般都是**bug**或者**缺陷**，我们不必把所有的细节都解释给开发者，但是必要的信息是要提供的，比如一个简单的错误文本。

举个例子，我们正在写一个包，其中有一个用于发送（大陆）短信的方法：

```go
// golang
import (
    "regexp"
    "github.com/pkg/errors"
)
var (
    phoneRegexp = regexp.MustCompile("^((\\+86)|(86))?\\d{11}$")
    ErrPhoneSmsExceedLimit = errors.New("target phone exceed send limits")
)
func SendSms(phone string, content string) error {
    if phone == "" {
        return errors.New("phone is required")
    }
    if content == "" {
        return errors.New("content is required")
    }

    if !phoneRegexp.MatchString(phone) {
        return errors.New("phone format incorrect")
    }

    if exceedLimits(phone) {
        return ErrPhoneSmsExceedLimit
    }
    // ...
}
```

由于使用`SendSms`的人只可能是开发者，所以简单的将错误信息返回就可以了，无须再多做处理。

这里需要插一句，一切的错误都会影响消费方的执行（除非消费方总是忽略错误），所以总在某个地方将我们返回的错误展示给开发者。

在上面这个例子中，我们已经要求了`phone`和`content`不应该为空字符串，那么消费方为什么还要给我空字符串呢？**这是 bug**。

另外，如果手机号超过了每日发送的条数限制，这**不是 bug**，而是业务错误，所以我们用`ErrPhoneSmsExceedLimit`提醒开发者，需要额外留意和处理一下，必要的时候用一些友好信息告诉用户。在该例子中是假定`SendSms`和消费方处于同一进程，所以只需要通过判断`err == sms.ErrPhoneSmsExceedLimit`就可以准确的捕获到业务错误。那如果这个发短信的方法在一个微服务之后呢？上面我们也提到了，这时候需要有某种标记：

```go
// golang
var ErrPhoneSmsExceedLimit = NewBusinessError("310001", "target phone exceed send limits")
func SendSms(phone string, content string) error {
    // ...
    if exceedLimits(phone) {
        return ErrPhoneSmsExceedLimit
    }
    // ...
}
```

是不是殊途同归了？当然了，这其中还涉及到一些边界上对错误的包装与转换，我们在后面会提到。

那么接下来，如果这个方法还需要调用一些别的`RPC`（这里假定是个`Restful`服务）才能完成最终的发送，并且调用有可能会有错误，该怎么处理呢？我们会包装它：

```go
// golang
func SendSms(phone string, content string) error {
    // ...

    provider := service.NewSmsProvider("appid", "appsecret")
    res, err := provider.Send(phone, content)
    if err != nil {
        return errors.Wrapf(err, "send sms to phone %s failed", phone)
    }

    // ...
}
```

如此，消费方看到的只是`send sms to phone xxx failed`（包装进去的低层`err`会在边界处切掉），不过不影响我们服务本身打印出调用栈，方便我们知道是我们使用`RPC`的姿势有问题，还是网络出现故障了，还是……总之，我们进行不下去了。我们不必告诉消费方这些低层的错误细节，但是我们需要保留这些细节方便自己。

我们继续思考，如果调用`RPC`成功返回了，就一定代表成功了吗？当然不是，没有`err`很可能只是说明整个`RPC`成功完成，但没说业务一定是成功的呀，所以我们还得对`res`进一步分析：

```go
// golang
func SendSms(phone string, content string) error {
    // ...
    res, err := provider.Send(phone, content)
    if err != nil {
        // ...
    }

    switch res.Code {
        case "0000":
            return nil
        case "1001":
            log.Printf("sms provider report [%s] insufficient balance", res.code)
        default:
            log.Printf("sms provider report [%s] %s", res.Code, res.Msg)
    }
    return errors.New("send sms failed")
}
```

我们已知的业务码只有`0000`代表成功，所以返回`nil`表示本次调用成功；`1001`代表余额不足，其它的我们可能并不关心，那么在简单的记录日志之后，返回给调用方的只有`send sms failed`。这是因为，我的错误我知道，我依赖服务的错误我也应该知道，但是，依赖我的服务如果不是使用姿势不对，或者业务不正确的话，没有理由了解这背后发生的过多细节，唯一需要让消费方知道的就是**没成功**。与此同时，我们记录了所有的细节，不管是显式的`log.Printf`还是在边界上打印的调用栈，都将进一步帮助我们分析和修复错误，或者改善实现细节。

那么，如果此时`SendSms`方法还需要调用并处理另一个**内部**的方法`darkMagic(phone string) error`返回的错误呢？没关系，仍然`errors.Wrap(err, "cannot perform such operation")`就好了。这不仅仅是给调用方看，更重要的是，这说明了在`darkMagic`里**可能有一个 bug**，需要我们自己处理，因为，我们是最清楚这些逻辑的，如果一切检查（参数的，业务的）都没问题，还会在内部出错，那么就可能是我们的实现有问题了。好在，这一类的缺陷通过单元测试一般都可以检测出来。

> 一个小问题，`darkMagic()`里如果调用`spellForce()`又得到`error`了怎么办？
> 答案是，直接`return err`
> 堆栈信息在`spellForce()`扔出的`error`里就有了，错误信息也很明确，着实不用再包装一层。
> 也就是说，进程内遇到的`error`，只在离边界最近的地方才需要`errors.Wrap()`成对调用方友好（和隐藏细节）的`error`，其它的都直白的往上`return err`就好

总结一下：

- 你使用我的姿势不对，例如空字符串，会造成我的错误，直接返回`errors.New()`，这是**bug**，你去处理
- 你使用的姿势是对的，我定睛一看是业务上问题，给你一个让你有机会通过**错误类型**或者**错误码**知道的原因，你**酌情处理**
- 你使用的姿势是对的，我检查发现业务也没毛病，但是我依赖的一些服务（例如数据库）出幺蛾子了，那么我会`Wrap`成一个既方便我调查原因，同时在不让你关注过多细节的前提下告诉你：**失败了**，你**酌情处理**，例如重试或者告诉最终用户“我们的服务开了会小差，请稍后重试”等
- 如果我觉得这一定是个很严重的问题，并且我也无法解决，同时认为你也不该尝试解决，那么就`panic`吧。这一点在在线业务上几乎遇不到，除了“内存满了”、“堆栈爆了”这些无法抗拒的原因，`panic`的很少会有

## 可恢复故障如何处理

我们在“错误信息应该暴露多少”一节里已经展示过一些处理方式，尤其是对跨越多层边界的错误，进程内遇到错误的情形等。非边界处的错误处理很直白，上一节也做出了解释和示例，这一节我们讨论一下在边界处如何处理遇到的`error`。

所谓边界，就是离调用方最近的地方，调用方可以是某个服务，也可以是用户使用的某种客户端，总之是在消费你在边界处提供的服务。边界以内，只有进程内可见。

所以，我们可以认为，一个**用户微服务的`GetUserById()`**在边界上，一个`beego.Get("/",func(ctx *context.Context){})` 用`MVC`实现的方法也在边界上。

通常情况下，在边界处，我们就需要对下游产生的错误做出判断，同时，对一些非业务错误一些包装，隐藏错误细节。如果边界不是面向最终用户的，那么也会提供一些开发者友好的错误文本。

我们分别来这其中处理错误的不同。

### 面向非用户的边界

对于一个用户微服务的`GetUserById()`，它的消费方一般不会是最终用户，而是某种**聚合网关**或者其它**微服务**，所以它藏匿在整个安全壁垒之后。我们通常会这么处理：

```go
// golang
import (
    "context"
    "github.com/pkg/errors"
    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/bson/primitive"
    "go.mongodb.org/mongo-driver/mongo"
)
var ErrUserNotValid = NewBusinessError("500213", "user is not valid")
func GetUserById(userId string) (*model.User, error) {
    if userId == "" {
        return errors.New("userId is required")
    }

    uid, err := primitive.ObjectIDFromHex(userId)
    if err != nil {
        return nil, errors.Wrap(err, "userId format incorrect")
    }

    user := &model.User{}
    coll := db.Collection("users")

    if err := coll.FindOne(context.TODO(), bson.M{"_id": uid}).Decode(user); err != nil {
        if err == mongo.ErrNoDocuments {
            // maybe return nil, nil is fine
            // but, depends on design, be careful
        }

        return nil, errors.Wrap(err, "cannot perform such operation")
    }

    // maybe do local business check
    if localBusinessCheck(user) {
        return nil, ErrUserNotValid
    }

    // maybe call RPC to do business action
    fine, err := rpc.BusinessAction(user)
    if err != nil {
        // err usually wrapped in rpc particular message type
        // so we need abstract real error from wrapper type
        rpcStatus := rpc.Convert(err)

        if rpcStatus.Type == rpc.Status_Business_Error {
            code := rpcStatus.GetMeta("code")
            msg  := rpcStatus.GetMeta("msg")
            return nil, NewBusinessError(code, msg)
        }

        cause := rpcStatus.Error()
        return nil, errors.Wrap(cause, "service unavailable")
    }
    if !fine {
        return nil, ErrUserNotValid
    }

    return user, nil
}
```

这段示例很有意思。首先，如何处理下游支撑服务返回的异常？支撑服务（例如数据库、缓存、中间件等等）往往没有业务，它们返回的错误就是单纯的错误，需要开发者每时每刻关注和处理。所以，在这里直接包装并返回。于此同时，`GetUserById()`的消费方得到了只应该它们关注的`cannot perform such operation`，而在用户微服务里，我们得到了完整的调用栈和错误信息。

其次，本地的业务检查如果失败，我们将直接返回一个预定义好的`ErrUserNotValid`，表示一个业务上的失败。

最后，如果涉及进一步的远程`RPC`调用，事情会变的稍微麻烦一些。远程的`RPC`调用可能有错误，但是错误类型比较复杂。通过`RPC`的方式传递错误不如进程内调用那么简单直白，为了能够顺利序列化，很多的`RPC`框架都会将错误信息打包成为某种专有的结构，所以，我们需要一些手段从这些专有结构中提取出我们需要的信息出来。

> GRPC 会将错误打包成为`google.golang.org/genproto/googleapis/rpc/status`包中的`status.Status`结构，`status.Status`里包含了`Code`、`Message`、`Details`，我们通常可以约定`Code`为`10`代表业务错误（10 代表 Aborted），同时将业务码打包进`Details`里。

> GraphQL 也有类似的方式，在返回的数据中，除了包含正常数据的`data`字段外，还有一个`errors`数组字段。一般发生错误时，会通过`errors.[].message`提供错误信息供客户端使用，但当我们需要提供业务码信息时，这个字段显然不太适合使用。不过好在，除了`errors.[].message`，GraphQL 还提供了`errors.[].extensions`结构用于扩展错误信息。于是乎，可以和消费方约定一个业务码所使用的具体字段，例如`errors.[].extensions.code`，如此便很好的解决了问题。

> Restful 的方式其实很像是 GraphQL 的方式，由于`http`上不提供额外的序列化通道，能用的只有`body`这一个选项（用`header`？不能够！），所以看起来只能提供`{ "data": {}, "err_code": "", "err_msg": "" }`这样的万能包装。其实大可不必，没有错误的情况下，正常把数据写入`body`，当出现业务错误时，只要返回`{ "err_code": "", "err_msg": "" }`，**同时把 status code 设置为 400**即可，这样就能把万能的`data`字段解放出来了。如果是一般的错误，例如少参数、参数不允许为空等，这时候不用提供`err_code`，只提供`err_msg`，**同时把 status code 设置为 500**即可。一股脑的`200`真的不是什么好设计。

通过`rpc.Convert()`类似的工具函数，我们能从`RPC`的`error`中拿到原始的结构数据，然后通过判断，确定是否为业务上的错误（所代表的类型），进而将原始的业务错误重新向外扔出，不需要做额外的处理。如果不是业务上的错误，那么就是**bug**、缺陷或者传输级别的故障，我们仍旧可以通过包装扔出，留下堆栈和详细信息在微服务内。

这或多或少的需要一种**统一的设计和约定**，例如将`RPC`错误的类型字段的某个特定 key，约定好专门用于存放业务错误码，否则的话将无法区分“业务错误”和“其它错误”。

示例中关于`RPC`错误的代码稍显啰嗦，我们其实可以稍微重构一下：

```go
// golang
func handleRpcError(err error, wrapMsg string) error {
    if err == nil {
        return nil
    }

    rpcStatus := rpc.Convert(err)
    if rpcStatus.Type == rpc.Status_Business_Error {
        code := rpcStatus.GetMeta("code")
        msg  := rpcStatus.GetMeta("msg")
        return nil, NewBusinessError(code, msg)
    }

    cause := rpcStatus.Error()
    return nil, errors.Wrap(cause, wrapMsg)
}

// in pratice
func FindUserById(userId string) error {
    // ...
    fine, err := rpc.BusinessAction(user)
    if err != nil {
        return nil, handleRpcError(err, "service unavailable")
    }
}
```

那么，如果是更靠近最终用户的“边界”，又该如何处理呢？

### 面向用户的边界

很明确的就是，首先用户很大程度上是关心**业务码**的，至少用户使用的客户端是关心的；其次，用户是不关心什么连接字符串错误、`userId is required`等等这些错误的。所以，**业务错误需要明确给出，前置检查错误只给开发者，其它不可预料的错误全部简单转换为“服务当前不可用”**。

有几个简单的观点：

- 有业务码错误的才需要对用户显示信息，其它的一律可显示为视为**出错了，请稍后重试**
- 有业务码的，说明是非技术的错误，其他一切要么是**bug**，需要开发人员在上线前处理完毕，要么是运行错误，比如数据库异常。需要告诉用户的只有**出错了，请稍后重试**，不会也不能再告诉更多
- 身份证号格式不对，电话号格式不对，这种错误在严格意义上算是**bug**，应该在调用`API`前就检验好的。如果设计不那么严格，可以适当的返回业务码帮助一下，但也只是友情帮助，该客户端做的验证还是得做的

我们来看最后一个例子：

```go
// golang
import (
    "github.com/pkg/errors"
)
var ServiceUnavailableMessage = "service unavailable"

type LoginReq struct {
    Username string
    Password string
}

func Login(ctx context.Context, req LoginReq) (*model.Credential, error) {
    if req.Username == "" {
        return nil, errors.New("username is required")
    }

    if req.Password == "" {
        return nil, errors.New("password is required")
    }

    // FindByUsername
    // maybe got business error: '[10011] user doesn't exists'
    user, err := rpc.UserService.FindByUsername(req.Username)
    if err != nil {
        return handleRpcError(err, ServiceUnavailableMessage)
    }

    // SignIn
    // maybe got business error: '[20001] account is disabled'
    // maybe got business error: '[20002] password is incorrect'
    // maybe got business error: '[20003] login place abnormal'
    cred, err := rpc.AccountService.SignIn(user.Id, req.Password)
    if err != nil {
        return handleRpcError(err, ServiceUnavailableMessage)
    }

    credential := &model.Credential{}
    if err := credential.Load(cred); err != nil {
        return errors.Wrap(err, ServiceUnavailableMessage)
    }

    return credential, nil
}
```

这是非常常见的一种`API`服务的写法，我省去了一些不必要的细节，例如`Routing`或者`Response`相关的东西。其实和普通的微服务实现没有什么两样，除了几个小细节：

- 对参数的校验还是必要的，不能因为微服务校验过参数，消费方就不做校验了
- 除了参数校验的错误，仍然需要对下游服务返回的业务错误同步的向上返回
- 除了参数错误和业务错误，其它的错误会包装成`service unavailable`，不向用户泄露任何的技术细节

通常，在这种类型的服务中，会有一个类似中间件的东西，统一的处理一切的错误（或者，建议自己实现一个），或者叫全局的错误处理函数、生命周期钩子等等，总之在我们的`Login()`函数返回错误后，能够以统一的方式响应给用户端，那具体会是什么样呢？

```go
// golang
type UserError struct {
    ErrCode string `json:"err_code"`
    ErrMsg  string `json:"err_msg"`
}

func handleGlobalError(ctx HttpContext, err error) {
    if err != nil {
        if e, ok := err.(*BusinessError); ok {
            ue := &UserError{
                ErrCode: e.Code,
                ErrMsg:  e.Msg,
            }

            ctx.Response.WriteJson(ue)
            ctx.Response.SetStatus(400)
        } else {
            ue := &UserError{
                ErrMsg:  err.Error(),
            }

            ctx.Response.WriteJson(ue)
            ctx.Response.SetStatus(500)
        }
    }
}
```

当然，这个函数只是概念上的解释，具体到每一个不同的场景会有不同的`API`和方式。实际上，如果能够支持这种全局错误处理，那么`credential.Load(cred)`产生的错误实际都不用`Wrap`，只需在处理全局错误的时候，直接将非业务错误的`UserError`的`ErrMsg`设置成`service unavailable`就可以了，这也避免了处处都`errors.Wrap(err, ServiceUnavailableMessage)`，让简洁性更进一步。

如此，世界得以清静。

（完）
