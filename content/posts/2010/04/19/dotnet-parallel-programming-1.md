---
title: ".NET 4中的并行编程(上)"
date: 2010-04-19T00:00:00+08:00
draft: false
type: "post"
tags:
    - "C#"
    - ".NET"
    - "Parallel"
---

并行是.NET 4中新加入的特性，为了使程序在多核心多CPU环境运行的更好、更快、更强大。

并发和并行是不一样的，并发最多可以算做是多线程，而所谓并行是将任务分散到不同的CPU上同时执行。尤其值得我们关注的是，在Web环境下的先天并行特性，使得并行编程成为解决性能瓶颈的又一武器。

.NET 4中的并行编程主要是Parallel和Task，微软强势构建了TPL(Task Parallel Library)，使开发过程变得简洁。

接下来，先看一个简单的Demo：

```csharp
using System.Threading.Tasks;

Parallel.Invoke(
    () => {
        Thread.Sleep(1000);
        Console.WriteLine("1");
    },
    () => {
        Thread.Sleep(1000);
        Console.WriteLine("2");
    },
    () => {
        Thread.Sleep(1000);
        Console.WriteLine("3");
    });
```
`Parallel.Invoke()`方法可以接收一个Action委托的params数组。我们来猜一下上段代码的执行结果是什么？123的顺序还是132？213？231？这个，不一定，得看人品……呵呵，开个玩笑，并行执行时，将上面三个Lambda表达式传递的方法分别看做一个“任务”，将其分配至空闲CPU，然后执行，由于CPU的情况是不一定的，所以执行完毕的顺序也就有了差异。

由于很多需要并行执行的任务都有一定的相似性，所以一般情况下我们可以用一种类似for循环的方式对这些任务进行并行的处理，例如在.NET 4中可以这么做：

```csharp
Parallel.For(0, 10, i => {
    Console.WriteLine(i);
});
```
但样做有时却不方便，例如我们要处理一个集合中的数据，既然集合实现了IEnumerable接口，为什么还要用索引来遍历呢？

其实我们是可以方便并行的处理一个数据集合的，以在多CPU环境下获得更高的性能。具体的，我们可以这么做：

```csharp
var dataList = new string[] { "data1", "data2", "data3" };//IEnumerable interface.
Parallel.ForEach(dataList, taskOptions, data => {
    Console.WriteLine(data + " processing");
});
```
taskOptions是一个可选的参数，用于指定并行任务执行的方式，可以这样设置它：

ParallelOptions taskOptions = new ParallelOptions();
taskOptions.MaxDegreeOfParallelism = 2;
这里只是简单的设置了一下并行执行的程序，可以理解为使用的CPU核心数，-1表示由CLR来决定，因为我的本本是两核的，那就指定为2吧……

当然了，processing的顺序也不一定的顺序了，原因看上面。

Parallel为我们提供了一种最原始的控制并行任务的过程，但这在实际的使用中是远不够的，我们需要更加细腻的控制并行执行中的阶段和并发冲突情况(并行中当然也有冲突了)等，.NET 4为我们带来的Task很好的解决了这个需求。

先上一个简单的例子：
```csharp
Task task1 = new Task(() => {
    Console.WriteLine("Task1");
});
task1.Start();

Task task2 = Task.Factory.StartNew(() => {
    Console.WriteLine("Task2");
});
```
这里使用了两种方式来启动一个Task，第一种是普通的做法，先实例化一个Task，再调用Start()方法来启动一个任务。第二种是使用Task的工厂方法来初始化并启动。孰好孰坏就全看个人喜好了。

当我们的Task执行完毕后会产生一个新结果时，可以在初始化任务时指定返回的类型，然后访问.Result属性来获取任务结束后的值，例如下面：
```csharp
Task task1 = Task.Factory.StartNew(() =>{
    Thread.Sleep(1000);
    return "string1";
});

Task task2 = Task.Factory.StartNew(() =>{
    Thread.Sleep(1000);
    return "string2";
});

Task task3 = Task.Factory.StartNew(() =>{
    Thread.Sleep(1000);
    return "string3";
});

Console.WriteLine(task1.Result);
Console.WriteLine(task2.Result);
Console.WriteLine(task3.Result);
```
需要说明的是，当访问`Task.Result`属性时，如果对应的Task还没有结束，那么在访问线程将会阻塞，直到所对应的Task执行完毕才会继续。

这种阻塞当然是安全的，然而很多时候，我们需要确保某些任务结束后才可以执行后续的代码，这时候会怎么做呢？
```csharp
Task task1 = new Task(() => { Console.WriteLine("Task1"); });
Task task2 = new Task(() => { Console.WriteLine("Task2"); });
Task task3 = new Task(() => { Console.WriteLine("Task3"); });

task3.Start();
Task.WaitAll(task3);//params Task

Console.WriteLine("Task3 complete");

Task.WaitAny(task3, task2);//task2并没有运行
Console.WriteLine("task2和task3中必然完成了一个");
task1.Start();
```
`Task.WaitAll()`方法会阻塞线程直到指定的Task全部完成后才会向下运行，这相当于设了一道关卡，确保指定任务全部结束。

而`Task.WaitAny()`方法会检测指定的全部Task，只要其中的任何一个Task完成，那么就可以继续。

其实在多线程中也有类似的做法，并不是并行的专利，例如这样：

```csharp
CountdownEvent cde = new CountdownEvent(3);

ThreadPool.QueueUserWorkItem(_ =>{
    Thread.Sleep(2000);
    Console.WriteLine("Work item 1");
    cde.Signal();
});
ThreadPool.QueueUserWorkItem(_ =>{
    Thread.Sleep(4000);
    Console.WriteLine("Work item 2");
    cde.Signal();
});
ThreadPool.QueueUserWorkItem(_ =>{
    Thread.Sleep(6000);
    Console.WriteLine("Work item 3");
    cde.Signal();
});

cde.Wait();
Console.WriteLine("All complete.");
```
只不过Task的方法更加的直接而已。

然而任务间有时也有着依赖关系，例如Task1依赖于Task2的执行，Task2又依赖于Task3的执行，这样，完全可以用普通的单线程来替代了，为什么还要使用并行呢？其实就拿上面的例子来说，Task3在一个CPU1上执行完毕了Task2在CPU2上立即启动，这时CPU1完全可以用来做别的并行任务了，不是么？这就是任务链的作用，我们可以用一个Demo来说明：

```csharp
Task task1 = new Task(() =>{
    Console.WriteLine("Task1");
    return 1;
});
Task task2 = task1.ContinueWith(parent => {
    Console.WriteLine("Task2");
    Console.WriteLine("Task1's result:{0}", parent.Result);
    return "Task2";
});
Task task3 = task2.ContinueWith(parent => {
    Console.WriteLine("Task3");
    Console.WriteLine("Task2's result:{0}", parent.Result);
    return "Task3";
});

task1.Start();
```
当调用`task1.Start()`方法后task1启动，执行完毕后task2启动，再然后task3启动，呈一条链状执行，可以写在一条语句：

```csharp
Task task = Task.Factory.StartNew(() => {
    Console.WriteLine("Task1");
    return 1;
}).ContinueWith(parent => {
    Console.WriteLine("Task2");
    Console.WriteLine("Task1's result:{0}", parent.Result);
    return "Task2";
}).ContinueWith(parent => {
    Console.WriteLine("Task3");
    Console.WriteLine("Task2's result:{0}", parent.Result);
    return "Task3";
});
```
显然，这些只能在任务的外部进行并行执行阶段的控制，如果想要更加细度的控制并行的执行，例如在并行过程的内部实现阶段的控制，那么就需要引入Barrier结构了。Barrier结构可以为并行过程内部提供“阶段关卡”，让并行也分阶段来协作，一个简单的例子是这样：

```csharp
Barrier barrierDemo = new Barrier(3, (barrier) => {
    Console.WriteLine("Phase {0} has been completed.", barrier.CurrentPhaseNumber + 1);
});

Task task1 = Task.Factory.StartNew(() => {
    Console.WriteLine("Task1,phase 1.");
    barrierDemo.SignalAndWait();// phase 1

    Console.WriteLine("Task1,phase 2.");
    barrierDemo.SignalAndWait();// phase 2

    Console.WriteLine("Task1,phase 3.");
    barrierDemo.SignalAndWait();// phase 3
});
Task task2 = Task.Factory.StartNew(() => {
    Console.WriteLine("Task2,phase 1.");
    barrierDemo.SignalAndWait();// phase 1
    Console.WriteLine("Task2,phase 2.");
    barrierDemo.SignalAndWait();// phase 2
    Console.WriteLine("Task2,phase 3.");
    barrierDemo.SignalAndWait();// phase 3
});
Task task3 = Task.Factory.StartNew(() => {
    Console.WriteLine("Task3,phase 1.");
    barrierDemo.SignalAndWait();// phase 1

    Console.WriteLine("Task3,phase 2.");

    barrierDemo.SignalAndWait();// phase 2
    Console.WriteLine("Task3,phase 3.");

    barrierDemo.SignalAndWait();// phase 3
});

Task.WaitAll(task1, task2, task3);
Console.WriteLine("All complete.");
```

三个Task是并行执行的当然没有问题，因而顺序也就不一定，原因再请看上面。我们需要解决的问题是怎样把Task分为3个阶段，让所有的Task在对应的阶段能统一一下步子，然后再往下走，Barrier就是为此而生，先在主线线程内声明一个Barrier实例并指定需要控制的阶段数量，然后在Task内分别来使用SignalAndWait()来设阶段关卡，可以预想执行的结果是：

```
Task2,phase 1.    
Task1,phase 1.    
Task3,phase 1.
Phase 1 has been completed. 
Task1,phase 2. 
Task2,phase 2. 
Task3,phase 2. 
Phase 2 has been completed. 
Task3,phase 3. 
Task1,phase 3. 
Task2,phase 3.  
Phase 3 has been completed.
All complete.
```
当然，每个阶段的完成顺序可以是不一样的。

下一次将要讨论的是并行编程中的关于异常处理、访问安全问题以及PLINQ，当然了，这些都只是一些开头，当真正要在工程中使用并行，需要考虑的问题还有很多，毕竟，不管是多线程还是并行都提升了代码的复杂度，我们必须要衡量在大的环境下这样做是否值得……