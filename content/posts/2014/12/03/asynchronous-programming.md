---
title: "使用异步编程"
date: 2014-12-03T00:00:00+08:00
draft: false
type: "post"
tags:
    - "Asynchronous"
    - "JavaScript"
    - "C#"
---
## 导言

现代的应用程序面临着诸多的挑战，如何构建具有可伸缩性和高性能的应用成为越来越多软件开发者思考的问题。随着应用规模的不断增大，业务复杂性的增长以及实时处理需求的增加，开发者不断尝试榨取硬件资源、优化。

在不断的探索中，出现了很多简化场景的工具，比如提供可伸缩计算资源的 Amazon S3[^amazon-s3]、Windows Azure[^win-azure]，针对大数据的数据挖掘工具 MapReduce[^map-reduce]，各种`CDN`服务，`云存储`服务等等。还有很多的工程实践例如敏捷[^agile]、DDD[^ddd]等提供了指导。可以看到，将每个关注层面以服务的方式提供，成为了越来越流行的一种模式，或许我们可以激进的认为，这就是 SOA[^soa]。

开发者需要将不同的资源粘合在一起来提供最终的应用，这就需要协调不同的资源。

<!--more-->

我们可以设想一个大的场景，开发者正在开发的一个用例会从用户的浏览器接收到请求，该请求会先从一个开放主机服务(OHS)获取必要的资源 res1，然后调用本机的服务 s1 对资源 res1 进行适应的转换产生资源 res2，接着以 res2 为参数调用远程的数据仓库服务 rs1 获取业务数据 bs1，最后以 bs1 为参数调用本机的计算服务 calc 并经过 10s 产生最终的数据。

简单的用 ASP.NET MVC 5 表示就是这样的（这些代码是我瞎掰的）：

```csharp
// notes: ASP.NET vNext changed MVC 5 usage,
// ActionResult now became IActionResult
public IActionResult CrazyCase(UserData userData) {
    var ticket = CrazyApplication.Ticket;

    var ohsFactory = new OpenHostServiceFactory(ticket);
    var ohs = ohsFactory.CreateService();

    var ohsAdapter = new OhsAdapter(userData);

    var rs1 = ohs.RetrieveResource(ohsAdapter);
    var rs2 = _localConvertingService.Unitize(rs1);
    var bs1 = _remoteRepository.LoadBusinessData(rs2);
    var result = _calculationService.DoCalculation(bs1);

    return View(result);
}
```

这可能是中等复杂度的一个场景，但是相信开发者已经意识到了这其中所涉及的复杂度。我们看到每一步都是依赖于前者所产生的数据，在这样一种场景之下，传统的多线程技术将极度受限，并且最顶层的协调服务将始终占用一个线程来协调每一步。

线程是要增加开销的，尤其是上下文的转换，别扯什么线程池了，创建线程的开销是节省了，上下文切换的开销才是要命的。

> 经济不景气，能省点儿资源就省点儿吧。

---

所以我们该怎么办？纵向扩展给服务器加多点内存？横向扩展上负载均衡？别闹了我们又不是民工，想问题不要太简单粗暴。解决的办法就是，异步，而且我们这篇也只讨论异步这一种技术。

## 为什么使用异步

那么，异步的优势在哪里？这首先要和同步做一个对比。

还是开头那个场景，示例代码所展示的是使用同步阻塞的方式来一步一步的执行，如下示意：

```
main) +++$----$------$--------$----------$+++
         |   /|     /|       /|         /
ohs )    $++$ |    / |      / |        /
              |   /  |     /  |       /
rs1 )         $++$   |    /   |      /
                     |   /    |     /
s1  )                $++$     |    /
                              |   /
calc)                         $++$

notes:
$ code point
+ thread busy
- thread blocked(means, wasted)
```

可以明显的看到，当主线程发起各个 service 请求后，完全处于闲置占用的状态，所做的无非是协调任务间的依赖顺序。这里所说的占用，其实就是 CPU 的时间片。

我们为什么要等所有的子任务结束？因为任务间有先后顺序依赖。有没有更好的方式来规避等待所带来的损耗呢？考虑一个场景，正上着班呢，突然想起要在网上买个东西，那么打开京东你就顺利的下单了，事情并没有结束，你不会等快递的小哥给你送来东西以后再接着今天的工作吧？你会给快递留下你的联系方式，让他**到了给你打电话**(耗时的 I/O 任务)，然后你继续今天烧脑的编程任务(CPU 密集型)。从人类的角度来看，这一定是最正常不过的，也就是要讨论的异步的方式。

> 一定有人会提议单开一个线程做收快递的任务，我同意这是一种解决方案，但是如果用等效的人类角度的语言来说，就是你将大脑的资源分成了两半，一半在烧脑编程，一半在盯着手机发呆，脑利用率下降太明显。而用异步的方式，你不需要关注手机，因为手机响了你就自然得到了通知。
> 当然，你也可以任性的说，我就喜欢等快递来了再干活。if so，我们就不要做朋友了。

所以我们可以有一个推论：异步所解决的，就是节省低速的 IO 所阻塞的 CPU 计算时间。

转换一下思路，我们使用异步非阻塞的方式来构建这段业务，并借助异步思想早已深入人心的`javascript`语言来解释，可以是这样的：

```javascript
// express

var ohs = require('./anticorruption/OpenHostService');
var localConvertingService = require('./services/LocalConverting');
var remoteRepository = require('./repositories/BusinessData');
var calculationService = require('./services/Calculation');

function(req, res) {
    var userData = req.body;

    // level1 nest
    ohs.retrieveResource(userData, function(err, rs1) {
        if(err) {
            // error handling
        }
        // level2 nest
        localConvertingService.unitize(rs1, function(err, rs2) {
            if(err) {
                // error handling
            }
            //level3 nest
            remoteRepository.loadBusinessData(rs2, function(err, bs1) {
                if(err) {
                    // error handling
                }
                //level4 nest
                calculationService.doCalculation(bs1, function(err, result) {
                    if(err) {
                        // error handling
                    }
                    res.view(result);
                });
            });
        });
    });
}
```

看着一层又一层的花括号也是醉了，我们之后会讨论如何解嵌套。那么这段代码所反应的是怎样的事实呢？如下示意：

```
main) +++$                           $+++
          \                         /
ohs )      $++$                    /
               \                  /
rs1 )           $++$             /
                    \           /
s1  )                $++$      /
                         \    /
calc)                     $++$

notes:
$ code point
+ thread busy
- thread blocked(means, wasted)
```

由于异步解放了原始的工作线程，使 CPU 资源可以不被线程的阻塞而被浪费，从而可以有效的提高吞吐率。

## 异步的使用场景

> 技术和选择和使用场景有着很大的关系，每项技术不都是银弹，使用对的工具/技术解决对的问题是开发者的义务。

开发者最多关注的是计算密集和 I/O 密集这两个维度，对于这两个维度往往有着不同的技术选型。

###计算密集型应用
何为计算密集型应用？下面两个人畜皆知的函数都是计算密集型的。

```fsharp
// F#
let fibonacci n =
    let rec f a b n =
        match n with
        | 0 -> a
        | 1 -> b
        | n -> (f b (a + b) (n - 1))
    f 0 1 n

let rec factorial n =
    match n with
    | 0 -> 1
    | n -> n * factorial (n - 1)
```

尤其是第二个阶乘函数，如果在调用的时候不小心手抖多加了几个 0，基本上可以出去喝个咖啡谈谈理想聊聊人生玩一天再回来看看有没有算完了。

简而言之，计算密集型的任务是典型的重度依赖 CPU/GPU，不涉及磁盘、网络、输入输出的任务。游戏中场景渲染是计算密集的，MapReduce 中的`Reduce`部分是计算密集的，视频处理软件的实时渲染是计算密集的，等等。

在这样的场景之下，异步是没有太大的优势的，因为计算资源就那么多，不增不减，用多线程也好用异步流也好，CPU 永远处于高负荷状态，这病不能治，解决方案只能是：

- 横向的集群方案
- 纵向的升级主机 CPU 或采用更快的 GPU
- 优化算法，使之空间/时间成本降低

但是有一种场景是可以考虑使用异步的，考虑一个分布式的计算场，一个计算任务发起后，协调者需要等待所有的计算节点子结果集返回后者能做最后的结果化简。那么此时，虽然场景是计算密集的，但是由于涉及到任务的依赖协调，采用异步的方式，可以避免等待节点返回结果时的阻塞，也可以避免多线程方式的上下文切换开销，要知道在这样的场景下，上下文切换的开销是可以大的惊人的。

相似的场景还有，一个桌面应用，假设点击界面上一个按钮之后会进行大量的计算，如果采用同步阻塞的方式，那么当计算完成之前 UI 是完全阻塞的跟假死一样，但是如何使用异步的方式，则不会发生 UI 阻塞，计算在结束后会以异步的方式来更新界面。还记得 WinForm 编程中的`BeginInvoke`和`EndInvoke`吗？虽然它们的实现方式是以单独线程的方式来实现异步操作的，但是这仍然属于异步流控制的范畴。

> 异步的实现方式有很多，可以使用已有的线程技术(Rx 和 C#的 async/await 就是使用这种方式)，也可以使用类似于 libuv 之类的 I/O 异步封装配合事件驱动(node 就是使用这种方式)。并于异步流控制的部分我们之后会讨论。

所以如果你的应用是计算密集型的，在充分分析场景的前提下可以适当的采用异步的方式。大部分的计算密集型场景是不用介入异步控制技术的，除非它可以显著改善应用的流程控制能力。

###I/O 密集型应用
何为 I/O 密集型应用？Web 服务器天然就是 I/O 密集型的，因为有着高并发量与网络吞吐。文件服务器和 CDN 是 I/O 密集型的，因为高网络吞吐高磁盘访问量。数据库是 I/O 密集型的，涉及磁盘的访问及网络访问。说到底，一切和输入输出相关的场景都是 I/O 密集型的。

I/O 囊括的方面主要是两方面：

- 网络访问
- 磁盘读写

简单粗暴的解释，就是接在主板南桥上的设备的访问都属于 I/O。多提一句，内存是直接接在北桥上的，这货，快。

开发者遇到最多的场景便是 Web 应用和数据库的高并发访问。其它的服务调用都属于网络 I/O，可归为一类。

典型的就是 Web 服务器接收到了 HTTP 请求，然后具体的 Web 框架会单开一个线程服务这个请求。因为 HTTP 是构建在 TCP 之上的，所以在请求结束返回结果之前，socket 并没有关闭，在 windows 系统上这就是一个句柄，在\*nix 之类的 posix 系统上这就是一个文件描述符，都是系统资源紧张的很。这是硬性的限制，能打开多少取决与内存与操作系统，我们暂且不关注这部分。该线程如果采用同步的方式，那么它程的生命周期会吻合 socket 的生命周期，期间不管是访问文件系统花了 10s 导致 cpu 空闲 10s 的时间片，还是访问数据库有 3s 的时间片空隙，这个线程都不会释放，就是说，这个线程是专属的，即便是使用线程池技术，该占还得占。

这有点像是银行的 VIP 专线，服务人员就那么多，如果每人服务一个 VIP 且甭管人家在聊人生聊理想还是默默注视，后面人就算是 VIP 也得等着，因为没人可以服务你了。

那么我们继续深入，线程也是一种相对昂贵的资源，虽然比创建进程快了太多，但是仍然有限制。windows 的 32 位操作系统默认每进程可使用 2GB 用户态内存（64bit 是 8Tb 用户态内存, LoL），每个线程有 1Mb 的栈空间（能改，但不建议。）；\*nix 下是 8Mb 栈空间，32 位的进程空间是 4Gb，64 位则大到几近没有用户态内存限制。我们可以假定 32 位系统下一个合理的单进程线程数量：1500。那么一个进程最大的并发量就是 1500 请求了，抛开多核不谈，这 1500 个线程就算轮班倒，并发量不会再上去了，因为一个 socket 一个线程。如果每个请求都是 web 服务器处理 1s 加访问数据库服务器 3s，那么时钟浪费率则大的惊人。况且，1500 个线程的上下文切换想想都是开心，开了又开[^apple-funny]。

不幸的是，之前的 web 服务器都是这么干的。此时我们思考，如果采用异步的方式，那 3s 的阻塞完全可以规避，从而使线程轮转的更快，因为 1s 的处理时间结束后线程返回线程池然后服务于另一个请求，从而整体提高服务器的吞率。

> 事实上，node 压根就没有多线程的概念，使用事件循环配合异步 I/O，一个线程总够你甩传统的 Web 服务器吞吐量几条街。没错，请叫我 node 雷锋。

再继续深入异步编程前，我们先理一理几个经常混淆的概念。

## 一些概念的区别

### 多核与多线程

多核是一种物理上的概念，即指主机所拥有的物理 CPU 核心数量，`总核心数 = CPU个数 * 每个CPU的核心数`。每个核心是独立的，可以同时服务于不同的进程/线程。

多线程是一种操作系统上的概念，单个进程可能创建多个线程来达到细粒度进行流程控制的目的。操作系统的核心态调度进程与线程，在用户态之下其实还可以对单个线程有更细粒度的控制，这称之为`协程（coroutine）`或`纤程（fibers）`。

多线程是指在单个进程空间内通过操作系统的调度来达到多流程同时执行的一种机制，当然，单个 CPU 核心在单位时间内永远都只是执行一个线程的指令，所以需要以小的时间片段雨露均沾的执行每个线程的部分指令。在切换线程时是有上下文的切换的，包括寄存器的保存/还原，线程堆栈的保存/还原，这就是开销。

### 并行与并发

关于并行，真相只有一个，单个 CPU 核心在单位时间内只能执行一个线程的指令，所以如果总核心数为 20，那么我们可以认为该主机的并行能力为 20，但是用户态的并行能力是要比这个低的，因为操作系统服务和其它软件也是要用 cpu 的，因此这个数值是达不到的。

一个题外话，如果并行能力为 20，那么我们可以粗略的认为，该主机一次可以同时执行 20 个线程，如果程序的线程使用率健康的话，保持线程池为 20 左右的大小可以做到完全的线程并行执行没有上下文切换。

那么并发则关注于应用的处理能力。这是一个更加侧重网络请求/服务响应能力的概念，可以理解为单位时间内可以同时接纳并处理用户请求的能力。它和多少 CPU 没有必然的关系，单纯的考量了服务器的响应回复能力。

### 阻塞与非阻塞

阻塞/非阻塞与同步/异步是经常被混淆的。同步/异步其实在说事件的执行顺序，阻塞/非阻塞是指做一件事能不能立即返回。

我们举个去 KFC 点餐的例子。点完餐交完钱了，会有这么几种情况：

- 服务人员直接把东西给我，因为之前已经做好了，所以能马上给我，这叫做非阻塞，我不需要等，结果立即返回。这整个过程是同步完成的。
- 服务人员一看没有现成的东西了，跑去现做，那么我就在这儿一直等，没刷微信没做别的干等，等到做出来拿走，这叫阻塞，因为我傻到等结果返回再离开点餐台。这整个过程是同步完成的。
- 服务人员一看没有现成的东西了，跑去现做，并告诉我说：先去做别的，做好了我叫你的号。于是我开心的找了个座位刷微信，等叫到了我的号了取回来。这叫做非阻塞，整个过程是异步的，因为我还刷了微信思考了人生。

异步是非阻塞的，但是同步可以是阻塞的也可以是非阻塞的，取决于消费的资源。

## 异步编程的挑战

异步编程的主要困难在于，构建程序的执行逻辑时是非线性的，这需要将任务流分解成很多小的步骤，再通过异步回调函数的形式组合起来。在异步大行其道的`javascript`界经常可以看到很多层的`});`，简单酸爽到妙不可言。这一节将讨论一些常用的处理异步的技术手段。

### 回调函数地狱

开头的那个例子使用了 4 层的嵌套回调函数，如果流程更加复杂的话，还需要嵌套更多，这不是一个好的实践。而且以回调的方式组织流程，在视觉上并不是很直白，我们需要更加优雅的方式来解耦和组织异步流。

使用传统的`javascript`技术，可以展平回调层次，例如我们可以改写之前的例子：

```javascript
var ohs = require('./anticorruption/OpenHostService');
var localConvertingService = require('./services/LocalConverting');
var remoteRepository = require('./repositories/BusinessData');
var calculationService = require('./services/Calculation');

function(req, res) {
    var userData = req.body;

    ohs.retrieveResource(userData, ohsCb);

    function ohsCb(err, rs1) {
        if(err) {
            // error handling
        }
        localConvertingService.unitize(rs1, convertingCb);
    }

    function convertingCb(err, rs2) {
        if(err) {
            // error handling
        }
        remoteRepository.loadBusinessData(rs2, loadDataCb);
    }

    function loadDataCb(err, bs1) {
        if(err) {
            // error handling
        }
        calculationService.doCalculation(bs1 , calclationCb);
    }

    function calclationCb(err, result) {
        if(err) {
            // error handling
        }
        res.view(result);
    }
}
```

解嵌套的关键在于如何处理函数作用域，之后金字塔厄运迎刃而解。

还有一种更为优雅的`javascript`回调函数处理方式，可以参考后面的`Promise`部分。

而对于像`C#`之类的内建异步支持的语言，那么上述问题更加的不是问题，例如：

```csharp
public async IActionResult CrazyCase(UserData userData) {
    var ticket = CrazyApplication.Ticket;

    var ohsFactory = new OpenHostServiceFactory(ticket);
    var ohs = ohsFactory.CreateService();

    var ohsAdapter = new OhsAdapter(userData);

    var rs1 = await ohs.RetrieveResource(ohsAdapter);
    var rs2 = await _localConvertingService.Unitize(rs1);
    var bs1 = await _remoteRepository.LoadBusinessData(rs2);
    var result = await _calculationService.DoCalculation(bs1);

    return View(result);
}
```

`async/await`这糖简直不能更甜了，其它`C#`的编译器还是生成了使用`TPL`特性的代码来做异步，说白了就是一些`Task<T>`在做后台的任务，当遇到`async/await`关键字后，编译器将该方法编译为状态机，所以该方法就可以在`await`的地方挂起和恢复了。整个的开发体验几乎完全是同步式的思维在做异步的事儿。后面有关于`TPL`的简单介绍。

### 异常处理

由于异步执行采用非阻塞的方式，所以当前的执行线程在调用后捕获不到异步执行栈，因此传统的异步处理将不再适用。举两个例子：

```csharp
try {
    Task.Factory.StartNew(() => {
        throw new InvalidOperationException("diablo coming.");
    });
} catch(InvalidOperationException e) {
    // nothing captured.
    throw;
}
```

---

```javascript
try {
  process.nextTick(function () {
    throw new Error("diablo coming.");
  });
} catch (e) {
  // nothing captured.
  throw e;
}
```

在这两个例子中，`try`语句块中的调用会立即返回，不会触发`catch`语句。那么如何在异步中处理异常呢？我们考虑异步执行结束后会触发回调函数，那么这便是处理异常的最佳地点。`node`的回调函数几乎总是接受一个错误作为其首个参数，例如：

```javascript
fs.readFile("file.txt", "utf-8", function (err, data) {});
```

其中的`err`是由异步任务本身产生的，这是一种自然的处理异步异常的方式。那么回到`C#`中，因为有了好用的`async/await`，我们可以使用同步式的思维来处理异常：

```csharp
try {
    await Task.Factory.StartNew(() => {
        throw new InvalidOperationException("diablo coming.");
    });
} catch(InvalidOperationException e) {
    // exception handling.
}
```

编译器所构建的状态机可以支持异常的处理，简直是强大到无与伦比。当然，对于`TPL`的处理也有其专属的支持，类似于`node`的处理方式：

```csharp
Task.Factory.StartNew(() => {
    throw new InvalidOperationException("diablo coming.");
})
.ContinueWith(parent => {
    var parentException = parent.Exception;
});
```

注意这里访问到的`parent.Exception`是一个`AggregateException`类型，对应的处理方式也较传统的异常处理也稍有不同：

```csharp
parentException.Handle(e => {
    if(e is InvalidOperationException) {
        // exception handling.
        return true;
    }

    return false;
});
```

### 异步流程控制

异步的技术也许明白了，但是遇到更复杂的异步场景呢？假设我们需要异步并行的将目录下的 3 个文件读出，全部完成后进行内容拼接，那么就需要更细粒度的流程控制。

我们可以借鉴 async.js[^asyncjs]这款优秀的异步流程控制库所带来的便捷。

```javascript
async.parallel(
  [
    function (callback) {
      fs.readFile("f1.txt", "utf-8", callback);
    },
    function (callback) {
      fs.readFile("f2.txt", "utf-8", callback);
    },
    function (callback) {
      fs.readFile("f3.txt", "utf-8", callback);
    },
  ],
  function (err, fileResults) {
    // concat the content of each files
  }
);
```

如果使用`C#`并配合`TPL`，那么这个场景可以这么实现：

```csharp
public async void AsyncDemo() {
    var files = new []{
        "f1.txt",
        "f2.txt",
        "f3.txt"
    };

    var tasks = files.Select(file => {
        return Task.Factory.StartNew(() => {
            return File.ReadAllText(file);
        });
    });

    await Task.WhenAll(tasks);

    var fileContents = tasks.Select(t => t.Result);

    // concat the content of each files
}
```

我们再回到我们开头遇到到的那个场景，可以使用`async.js`的`waterfall`来简化：

```javascript
var ohs = require('./anticorruption/OpenHostService');
var localConvertingService = require('./services/LocalConverting');
var remoteRepository = require('./repositories/BusinessData');
var calculationService = require('./services/Calculation');
var async = require('async');

function(req, res) {
    var userData = req.body;

    async.waterfall([
        function(callback) {
            ohs.retrieveResource(userData, function(err, rs1) {
                callback(err, rs1);
            });
        },
        function(rs1, callback) {
            localConvertingService.unitize(rs1, function(err, rs2) {
                callback(err, rs2);
            });
        },
        function(rs2, callback) {
            remoteRepository.loadBusinessData(rs2, function(err, bs1) {
                callback(err, bs1);
            });
        },
        function(bs1, callback) {
            calculationService.doCalculation(bs1, function(err, result) {
                callback(err, result);
            });
        }
    ],
    function(err, result) {
        if(err) {
            // error handling
        }
        res.view(result);
    });
}
```

如果需要处理前后无依赖的异步任务流可以使用`async.series()`来串行异步任务，例如先开电源再开热水器电源最后亮起红灯，并没有数据的依赖，但有先后的顺序。用法和之前的`parallel()`及`waterfall()`大同小异。另外还有优秀的轻量级方案 step[^step]，以及为`javascript`提供 monadic 扩展的 wind.js[^windjs]（特别像`C#`提供的方案），有兴趣可以深入了解。

### 反人类的编程思维

> 异步是反人类的

人类生活在一个充满异步事件的世界，但是开发者在构建应用时却遵循同步式思维，究其原因就是因为同步符合直觉，并且可以简化应用程序的构建。

究其深层原因，就是因为现实生活中我们是在演绎，并通过不同的`口头回调`来完成一系列的异步任务，我们会说你要是有空了来找我聊人生，货到了给我打电话，小红你写完文案了交给小明，小丽等所有的钱都到了通知小强……而在做开发时，我们是在列清单，我们的说法就是：我等着你有空然后开始聊人生，我等着货到了然后我就知道了，我等着小红文案写完了然后开始让她交给小明，我等着小丽确认所有的钱到了然后开始让她通知小强……

同步的思维可以简化编程的关注点，但是没有将流程进行现实化的切分，我们总是倾向于用同步阻塞的方式来将开发变成简单的步骤程序化，却忽视了用动态的视角以及消息/事件驱动的方式构建任务流程。

异步在编程看来是反人类的，但是从业务角度看却是再合理不过的了。通过当的工具及技术，使用异步并不是难以企及的，它可以使应用的资源利用更加的高效，让应用的响应性更上一个台阶。

## 扩展阅读

### Promise/Deferred

> 在一般情况下，Promise、Deferred、Future 这些词可以当做是同义词，描述的是同一件事情。

在`jQuery 1.5+`之后出现了一种新的 API 调用方式，相比于旧的 API，新的方式更好的解耦了关注点，并带来了更好的组合能力。

我们看一个传统的使用`ajax`的例子：

```javascript
$.get("/api/service1", {
  success: onSuccess,
  failure: onFailure,
  always: onAlways,
});
```

使用新的 API 后，调用的方式变成了：

```javascript
$.get("/api/service1").done(onSussess).fail(onFailure).always(onAlways);
```

`get`方法返回的是一个`promise`对象，表示这个方法会在未来某个时刻执行完毕。

`Promise`是 CommonJS[^commonjs]提出的规范，而`jQuery`的实现在其基础上有所扩展，旗舰级的实现可以参考 Kris Kowal[^kriskowal]的 Q.js[^qjs]。

我们使用`jQuery`来构建一个`promise`对象：

```javascript
var longTimeOperation = function () {
  var deferred = $.Deferred();

  // taste like setTimeout()
  process.nextTick(function () {
    // do operation.
    deferred.resolve();
    // if need error handling, use deferred.reject();
  });

  return deferred.promise();
};

$.when(longTimeOperation()).done(success).fail(failure);
```

由于`jQuery`生成的`Deferred`可以自由的进行`resolve()`和`reject()`，所以在返回时我们使用`.promise()`生成不含这个两方法的对象，从而更好的封装逻辑。

那么`Promise`究竟带给我们的便利是什么？`Promise`表示在未来这个任务会成功或失败，可以使用 1 和 0 来表示，那么开发者马上就开始欢呼了，给我布尔运算我能撬动地球！于是，我们可以写出如下的代码：

```javascript
$.when(uploadPromise, downloadPromise).done(function () {
  // do animation.
});
```

对于开头的那个例子我们说过有着更优雅的解回调函数嵌套的方案，那就是使用`promise`，我们来尝试改写开头的那个例子：

```javascript
var ohs = require('./anticorruption/OpenHostService');
var localConvertingService = require('./services/LocalConverting');
var remoteRepository = require('./repositories/BusinessData');
var calculationService = require('./services/Calculation');
var $ = require('jquery');

function(req, res) {
    var userData = req.body;

    function deferredCallback(deferred) {
        return function(err) {
            if(err) {
                deferred.reject(err);
            } else {
                var args = Array.prototype.slice.call(arguments, 1);
                deferred.resolve(args);
            }
        };
    }

    function makeDeferred(fn) {
        var deferred = $.Deferred();
        var callback = deferredCallback(deferred);
        fn(callback);
        return deferred.promise();
    }

    var retrieveResourcePromise = makeDeferred(function(callback) {
        ohs.retrieveResource(userData, callback);
    });

    var convertingPromise = makeDeferred(function(callback) {
        localConvertingService.unitize(rs1, callback);
    });

    var loadBusinessDataPromise = makeDeferred(function(callback) {
        remoteRepository.loadBusinessData(rs2, callback);
    });

    var calculationPromise = makeDeferred(function(callback) {
        calculationService.doCalculation(bs1 , callback);
    });

    var pipedPromise = retrieveResourcePromise
        .pipe(convertingPromise)
        .pipe(loadBusinessDataPromise)
        .pipe(calculationPromise);

    pipedPromise
        .done(function(result) {
            res.view(result);
        })
        .fail(function(err) {
            // error handling
        });
}
```

我们使用了一个高阶函数来生成可以兼容`deferred`构造的回调函数，进而使用`jQuery`的`pipe`特性(在`Q.js`里可以使用`then()`组合每个`promise`)，使解决方案优雅了很多，而这个工具函数在`Q.js`里直接提供，于是新的解决方案可以如下：

```javascript
var ohs = require('./anticorruption/OpenHostService');
var localConvertingService = require('./services/LocalConverting');
var remoteRepository = require('./repositories/BusinessData');
var calculationService = require('./services/Calculation');
var Q = require('q');

function(req, res) {
    var userData = req.body;

    var retrieveResourceFn = Q.denodeify(ohs.retrieveResource)
    var convertingFn = Q.denodeify(localConvertingService.unitize);
    var loadBusinessDataFn = Q.denodeify(remoteRepository.loadBusinessData);
    var calculationFn = Q.denodeify(calculationService.doCalculation);

    retrieveResourceFn(userData)
        .then(convertingFn)
        .then(loadBusinessDataFn)
        .then(calculationFn)
        .then(function(result) {
            res.view(result);
        }, function(err) {
            // error handling
        });
}
```

那我们如何看待`TPL`特性呢？我们看看`TPL`可以做什么：

- 以`Task`为基本构造单位，执行时不阻塞调用线程
- 每个`Task`是独立的，`Task`有不同的状态，可以使用`Task.Status`获取
- `Task`可以组合，使用类似`.ContinueWith(Task))`以及`.WhenAll(Task[])`、`.WhenAny(Task[])`的方式自由组合。

对比一下`Promise`：

- 以`Promise`为基本构造单位，表示一个将来完成的任务，调用时立即返回
- 每个`Promise`是独立的，`Promise`有不同的状态，可以使用`.state`获取
- `Promise`可以组合，使用`.then()`、`.pipe()`以及`.when()`来组合执行流程

可以看到，不论是`Promise`还是`TPL`，在设计上都有着惊人的相似性。我们有理由猜想在其它的的语言或平台都存在着类似的构造，因为异步说白了，就是让未来完成的事情自己触发后续的步骤。

### Pull vs. Push
在 GoF32[^gof32]中没有提到迭代器模式(Iterator)与观察者模式(Observer)的区别和联系，其实这两个模式有着千丝万缕的联系。

Iterator 反映的是一种 Pull 模型，数据通过同步的方式从生产者那里拉过来，我们通过它的定义便可看到这一事实：

```csharp
interface IEnumerator<out T>: IDisposable
{
    bool MoveNext();
    T Current { get; }
}
```

通过阻塞的方式调用`MoveNext()`，数据一个一个的拉取到本地。

而 Observer 反映的是一种 Push 模型，通过注册一个观察者（类似于回调函数），当生产者有数据时，主动的推送到观察者手里。观察者注册结束后，本地代码没有阻塞，推送数据的整个过程是异步执行的。我们通过它的定义来对比 Iterator：

```csharp
interface IObserver<in T>
{
    void OnCompleted();
    void OnError(Exception exception);
    void OnNext(T value);
}
```

我们发现，其实这两个接口是完全对偶的（参见 Erik Meijer[^erik-meijer]大大的论文 Subject/Observer is Dual to Iterator[^meijer2010]）：

- `MoveNext()`拉取下一个数据，`OnNext(T)`推送下一个数据
- `MoveNext()`返回值指示了有无剩余数据（完成与否），`OnCompleted()`指示了数据已完成（推送数据完成的消息）
- Iterator 是同步的，所以出现了异常直接在当前运行栈上，Observer 是异步的，所以需要另一种方式来通知发生了异常（参见上文中的异步处理一节），于是有了`OnError(Exception)`。

那么事情就变的有意思了，我们知道`Enumerable`的数据可以任意的方式组合，于是产生了像`LINQ`之类的库可供我们使用，但是这是一种阻塞的方式，因为 Iterator 本身就是一种 Pull 模型，这造就了同步等待的结果。

> 没错你是对的，如果使用 EF 之类的框架来查询数据库，大部分的操作是延迟执行的，表明操作并没有发生而是像占位符一样在那里。但是别忘了，你最终需要去查询数据库的，在查询的一刹那，世界还是阻塞的，等结果吧亲。

而 Observer 是异步 Push 的，有点像是事件驱动，有事件了触发，没事件了也不干扰订阅者的执行。

> 你是不是也隐隐的觉得事件也可以和 Push 模式一样有统一的模型？而且不只一次？

好，我们重复一遍：事件，非阻塞触发（并带有事件数据）。Push，非阻塞通知订阅者。

其实，这是同一种模式，语言中对事件（就是`event`关键字）的支持其实就是对 Observer 模式的支持，而`foreach`则实现了对 Iterator 模式的语言内建支持。所谓设计模式，就是因为语言的内建支持不够而出现的，说白了，是语言的补丁。

那么我们来看一看异常强大的 Rx[^rx]如何改变事件。

```csharp
// unitized event
var mouseDown = Observable
    .FromEventPattern<MouseEventArgs>(this.myPictureBox, "MouseDown")
    .Select(x =>x.EventArgs);

// unitized APM model
var request = WebRequest.Create("http://www.shinetechchina.com");
var webRequest = Observable
    .FromAsyncPattern<WebResponse>(request.BeginGetResponse, request.EndGetResponse);
```

最后我们看一个更加复杂的组合事件的例子，也就是之前一直讨论的异步流组合问题。Drag and Drop 这个场景做`Winform`的同学不会陌生，需要多少代码冷暖自知，如果借助`Rx`，那么事情就简单很多：

```csharp
var mouseDown = Observable
    .FromEventPattern<MouseEventArgs>(this.controlSource, "MouseDown")
    .Select(x => x.EventArgs.GetPosition(this));
var mouseUp = Observable
    .FromEventPattern<MouseEventArgs>(this.controlSource, "MouseUp")
    .Select(x => x.EventArgs.GetPosition(this));
var mouseMove = Observable
    .FromEventPattern<MouseEventArgs>(this.controlSource, "MouseMove")
    .Select(x => x.EventArgs.GetPosition(this));
var dragandDrop =
    from down in mouseDown
    from move in mouseMove.StartWith(down).TakeUntil(mouseUp)
    select new {
        X = move.X - down.X,
        Y = move.Y - down.Y
    };

dragandDrop.Subscribe(value =>
{
    DesktopCanvas.SetLeft(this.controlSource, value.X);
    DesktopCanvas.SetTop(this.controlSource, value.Y);
});
```

`Rx`也提供了`javascript`扩展，有兴趣可以深入研究。

（完）

[^agile]: http://agilemanifesto.org/
[^soa]: http://en.wikipedia.org/wiki/Service-oriented_architecture/
[^ddd]: http://www.domaindrivendesign.org/
[^amazon-s3]: http://aws.amazon.com/cn/s3/
[^win-azure]: http://azure.microsoft.com/
[^map-reduce]: http://research.google.com/archive/mapreduce.html
[^apple-funny]: http://www.zhihu.com/question/23544144
[^asyncjs]: https://github.com/caolan/async
[^step]: https://github.com/creationix/step
[^windjs]: https://github.com/JeffreyZhao/wind
[^commonjs]: http://wiki.commonjs.org/wiki/Promises/A
[^kriskowal]: https://github.com/kriskowal
[^qjs]: https://github.com/kriskowal/q
[^gof32]: http://en.wikipedia.org/wiki/Design_Patterns
[^erik-meijer]: http://en.wikipedia.org/wiki/Erik_Meijer_(computer_scientist)
[^meijer2010]: http://csl.stanford.edu/~christos/pldi2010.fit/meijer.duality.pdf
[^rx]: https://github.com/Reactive-Extensions/