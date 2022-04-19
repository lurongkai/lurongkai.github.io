---
title: ".NET 4中的并行编程(下)"
date: 2010-04-20T00:00:00+08:00
draft: false
type: "post"
tags:
    - "C#"
    - ".NET"
    - "Parallel"
---

上一次主要讨论了在.NET 4中如何编写并行程序，这次继续上次的话题。

当我们有能力使用前面所介绍的一些结构来构建我们的应用程序时，一个需要考虑的场景是：假如一个并行过程已经开始，在它没有完成前想取消它的话应该怎么做呢？其实这个问题很现实，在多线程程序中也会遇到，当然了，多线程编程时我们可以用Thread.Abort()来终结它，那么在并行中该如何实现呢？老规矩，上Demo：

```csharp
CancellationTokenSource tokenSource = new CancellationTokenSource();
CancellationToken token = tokenSource.Token;
Task task1 = Task.Factory.StartNew(() => {
    int i = 0;
    while (!token.IsCancellationRequested) {
        Thread.Sleep(500);
        Console.WriteLine("Task1,{0}", i++);
    }
}, token);
token.Register(() => {
    Console.WriteLine("Task1 has been canceled");
});

Console.ReadLine();
tokenSource.Cancel();
```
首先实例化一个CancellationTokenSource对象，这个对象用于管理并行过程中的取消动作。当创建一个Task的时候，传入一个CancellationToken对象，而这个对象可以由CancellationTokenSource.Token属性来获得。当Task开开始执行时，如果想取消这个Task，那么就可以调用CancellationTokenSource.Cancel()来取消与之相关的Task了。

其实，我们可以将CancellationTokenSource理解为一个犯罪团伙的头目，然后这个头目管理着CancellationToken小兵，然后当一个Task创建时将这个小兵安插在其中(无间道？)，当头目发指令Cancel()时，小兵在Task内部上演无间道，嗯……

然而，出来混，迟早要还的。Task能不能无异常的执行完毕还是个未知数。在.NET 4中，现在可以用AggregateException来处理这些异常了，它提供一种异常汇总机制，可以对未知的异常进行处理，上一个Demo：

```csharp
Task task1 = Task.Factory.StartNew(() => {
    Console.WriteLine("Task1 completed.");
});
Task task2 = Task.Factory.StartNew(() => {
    Console.WriteLine("Task2 processing.");
    throw new ArgumentOutOfRangeException("task2");
});

try {
    Task.WaitAll(task1, task2);
}
catch (AggregateException aggex) {
    aggex.Handle(ex => {
        Console.WriteLine(ex);
        return true;
    });
}
```

可以将具体的异常处理写在AggregateException.Handle方法内，Handle方法传入一个Func委托，用于处理具体的异常，例如上面，我们只是在Task内简单的抛出了一个ArgumentOutOfRangeException异常，当调用Task.WaitAll时异常被捕获，然后在aggex.Handle中具体处理。当然，可以将上述中传入的ex进行is判断来确定具体的异常，例如：if (ex is ArgumentOutOfRangeException) {…}。

聊完异常处理，接着聊访问安全。

访问安全的问题其实很显然，在多线程环境下要注意的问题无疑都是要在并行环境下考虑。这里，其实可以用线程安全来讲，因为不管是多线程还是并行，CPU调度的最小单位都是线程，所以，以下都用线程安全来说明。

要求线程安全，首先想到的肯定就是上锁。过去的一些手段就不说了，我们着重说一下.NET 4中的SpinLock互斥锁。

```csharp
SpinLock spinLock = new SpinLock();
bool locked = false;
try {
    spinLock.Enter(ref locked);
    // Do something.
}
finally {
    if (locked) {
        spinLock.Exit();
    }
}
```

其实不用解释也看的懂了。线程安全操作多用于对资源的访问控制上，例如IO，Stream等，而我们往往需要面对的其实是内存中的数据，例如集合，例如队列等。.NET 4为了并行，引入了一个新的命名空间：System.Collections.Concurrent;在这个空间之下有很多经过处理的基础结构，例如ConcurrentDictionary、ConcurrentQueue等，它们都是线程安全的，或者说是并行安全的，而且在使用方法上和见的System.Collections.Generic下的结构无异。接下来演示一个BlockingCollection结构的使用Demo：

```csharp
BlockingCollection<string> blockCollection = new BlockingCollection<string>();
ThreadPool.QueueUserWorkItem(o => {
    for (int i = 0; i < 200; i++) {
        blockCollection.Add("String" + i);
        Thread.Sleep(1000);
    }
});
ThreadPool.QueueUserWorkItem(o => {
    foreach (var i in blockCollection.GetConsumingEnumerable()) {
        Console.WriteLine("Read:{0}", i);
    }
});
```
我们用的是线程池的做法，当然，在并行编程中的做法也是一样的(有做法么？好像没有耶……)。可以看的出，Concurrent相关的结构可以大大简化并行编程中需要考虑的线程安全问题。

最后要讨论的是.NET 4中很NB的一个东西：PLINQ。并行并行，就是要多CPU协作同时执行，其实有理由相信多CPU可以提高查询的效率(没有说一定是DB查询，不涉及IO性能)，尤其是在内存中集合的查询，PLINQ就是Parallel化的LINQ，使我们的查询可以在多个CPU上同时执行，藉此提高查询效率。

要想在一个自定义的集合中实现LINQ功能，常用的做法就是实现IEnumerable，这样就可以使用LINQ的查询语法来实现类似SQL的“漂漂”代码，在.NET 4中要想实现一个能并行查询(PLINQ)的自定义集合，可以实现IParallelEnumerable，IParallelEnumerable继承于IEnumerable，实现起来其实也不困难。

怎样去用PLINQ呢？上一个Demo看看：

```csharp
var dataSet = new string[] { "data1", "data2", "data3", "data4" };
var results = from d in dataSet.AsParallel<string>()
                let result = d.ToUpper()
                select result;
foreach (var r in results) {
    Console.WriteLine(r);
}
```
只是简单的.AsParallel即可，很好很强大。我们可以使用.AsOrdered来实现在排序前进行缓存。

```csharp
var queryByOrder = from d in dataSet.AsParallel<string>().AsOrdered<string>()
                    orderby d descending
                    let result = d.ToUpper()
                    select result;
```
这次我们换个方面来将queryByOrder输出：

```csharp
queryByOrder.ForAll<string>(q => {
    Console.WriteLine(q);
});
```

看来扩展方法这块糖的确很好吃……

其实在普通的开发中，.AsParallel一下就OK了，我们来猜一下上面代码的结果是怎么样的？答案是不一定有顺序(即使在第二个示例中排序过)，详情请参见上篇中的解释。

通过两篇文章，我们讨论了一些.NET 4中并于并行开发的基础，在实际的开发中，是否选择并行仍然是一个有待商榷的问题，我们往往关心的是，并行究竟能为开发带来多大的复杂度，能为效率带来多大的提升，下一次我将对并行的效率进行讨论，欢迎大家一起加入。