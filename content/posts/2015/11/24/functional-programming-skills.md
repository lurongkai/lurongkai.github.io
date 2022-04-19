---
title: "函数式编程中的常用技巧"
date: 2015-11-24T00:00:00+08:00
draft: false
type: "post"
tags:
    - "Functional Programing"
    - "C#"
    - "F#"
    - "JavaScript"
---
在 Clojure、Haskell、Python、Ruby 这些语言越来越流行的今天，我们撇开其在数学纯度性上的不同，单从它们都拥有`一类函数`特性来讲，讨论函数式编程也显得很有意义。

一类函数为函数式编程打下了基础，虽然这并不能表示可以完整发挥函数式编程的优势，但是如果能掌握一些基础的函数式编程技巧，那么仍将对并行编程、声明性编程以及测试等方面提供新的思路。

很多开发者都有听过函数式编程，但更多是抱怨它太难，太碾压智商。的确，函数式编程中很多的概念理解起来都有一定的难度，最著名的莫过于[单子](https://en.wikipedia.org/wiki/Monad_(functional_programming)，但是通过一定的学习和实践会发现，函数式编程能让你站在一个更高的角度思考问题，并在某种层面上提升效率甚至是性能。我们都知道飞机比汽车难开，但是开飞机却明显比开汽车快，高学习成本的东西解决的大部分是高回报的需求，这不敢说是定论，但从实践来看这句话基本也正确。

<!--more-->

## 概述

[wikipedia](https://en.wikipedia.org/wiki/Functional_programming)上对于函数式编程的解释是这样的：

> In computer science, functional programming is a programming paradigm—a style of building the structure and elements of computer programs—that treats computation as the evaluation of mathematical functions and avoids changing-state and mutable data.

翻译过来是这样的：

> 在计算机科学中，函数式编程是一种编程范式，一种构建计算机结构和元素的风格，它将计算看作是对数学函数的求值，并避免改变状态以及可变数据。

关键的其实就两点：不可变数据以及函数求值（表达式求值）。由这两点引申出了一些重要的方面。

### 不变性

FP 中并没有变量的概念，东西一旦创建后就不能再变化，所以在 FP 中经常使用“值”这一术语而非“变量”。

不变性对程序并行化有着深远的影响，因为一切不可变意味着可以就地并行，不涉及竞态，也就没有了锁的概念。

不变性还对测试有了新的启发，函数的输入和输出不改变任何状态，于是我们可以随时使用 REPL 工具来测试函数，测试通过即可使用，不用担心行为的异常，不变性保证了该函数在任何地方都能以同样的方式工作。事实上，在函数式编程实践中，“编写函数、使用 REPL 工具测试，使用”三步曲有着强大的生产力。

不变性还对重构有了新的意义，因为它使得对函数的执行有了数学意义，于是乎重构本身成了对函数的化简。FP 使代码的分析变的容易，从而使重构的过程也变的轻松了许多。

### 声明性风格

FP 程序代码是一个描述期望结果的表达式，所以可以很轻松、安全的将这些表达式组合起来，在隐藏执行细节的同时隐藏复杂性。可组合性是 FP 程序的基本能力之一，所以要求每个组合子都有良好的语义，这和声明式风格不谋而合。

我们经常写`SQL`，它就是一种声明性的语言，声明性只提出`what to do`而不解决`how to do`的问题，例如下面：

```sql
SELECT id, amount
FROM orders
WHERE create_date > '2015-11-21'
ORDER BY create_date DESC
```

省去了具体的数据库查询细节，我们只需要告诉数据库要 orders 表里创建日期大于 11 月 21 号的数据，并只要 id 和 amout 两个字段，然后按创建日期降序。这是一种典型的声明性风格。

> 是的，我同意靠嘴是解决不了任何问题的，what to do 提出来后总得有地方或有人实现具体的细节，也就是说总是需要有 how to do 的部分来支持。但是换个思路，假如你每天都在写 foreach 语句来遍历某个集合数据，难道你没有想过你此时正在重复的 how to do 吗？就不能将某种通用的“思想”提取出来复用吗？假如你可以提取，那么你会发现，这个提取出来的词语（或函数名）已经是一种 what to do 层面的思想了。

再比如，对于一个整型数据集合，我们要通过 C#遍历并拿到所有的偶数，典型的命令式编程会这么做：

```csharp
// csharp
var result = new List<int>();
foreach(var item in sourceList) {
    if(item % 2 == 0) {
        result.Add(item);
    }
}
return result;
```

这对很多人来说都很轻松，因为就是在按照计算机的思维一步一步的指挥。那么声明性的风格呢？

```
// csharp
return sourceList.Where(item => item %2 == 0);
// or LINQ style
return from item in sourceList where item % 2 == 0 select item;
```

甚至更进一步，假设我们有声明性原语，可以做到更好：

```
// csharp
// if we already defined an atom function like below:
public bool NumberIsEven(int number) {
    return number % 2 == 0;
}

// then we can re-use it directly.
return sourceList.Where(NumberIsEven);
```

> 说句题外话，我有个数据库背景很深的 C#工程师同事，第一次见到 LINQ 时一脸不屑的说：C#的 LINQ 就是抄 SQL 的。其实我并没有告诉它 C#的 LINQ 借鉴的是 FP 的高阶函数以及 monad，只是和 SQL 长的比较像而已。当然我并不排除这可能是为了避免新的学习成本所以选用了和 SQL 相近的关键字，但是 LINQ 的启蒙却真的不是 SQL。

> 我更没有说 GC、闭包、高阶函数等先进的东西并不是.NET 抄 Java 或者谁抄谁，大家都是从 50 多年前的 LISP 以及 LISP 系的 Scheme 来抄。我似乎听到了 apple 指着 ms 说：你抄我的图形界面技术…

### 类型

在 FP 中，每个表达式都有对应的类型，这确保了表达式组合的正确性。表达式的类型可以是某种基元类型，可以是复合类型，当然，也可以是支持泛型类型的，例如 F#、ML、Haskell。类型也为编译时检查提供了基础，同时，也让屌炸天的类型推断有了根据。

F#的类型推断要比 C#强太多了，一方面是受益于 ML 及 OCamel 的影响，一方面是在 CLR 层面上泛型的良好设计。很多人并不知道 F#的历史可以追溯到.NET 第一个版本的发布（2002 年），而当时 F#作为一个研究项目，对泛型的需求很大，遗憾的是.NET 第一版并没有从 CLR 层面支持泛型。所以，F#团队参与设计了.NET 的泛型设计并加入到.NET 2.0 开始的后续版本，这也同时让所有.NET 语言获益。

那么我们以不同的视角审视一下泛型。何为泛型？泛型是一种代码重用的技术，它使用类型占位符来将真正的类型延迟到运行时再决定，类似一种类型模板，当需要的时候会插入真实的类型。我们换一个角度，将泛型理解为一种包装而非模板，它打包了某种具体的类型，使用类似 F#的签名表达会是这样：`'T -> M<'T>`，转变这种思维很重要，尤其是在编写 F#的计算表达式（即 Monad）时，经常会使用**包装类**这个术语。在 C#中也可以看到类似的方面，例如`int?`其实是指`Nullable<T>`对`int`类型的包装。

### 表达式求值

由于整个程序就是一个大的表达式，计算机在不断的求值这个表达式的同时也就意味着我们的程序正在运行。那么很有挑战的一方面就是，程序该如何组织？

FP 中没有语句的概念，就连常用的绑定值操作也是一个表达式而非语句。那么这一切如何实现呢？假设我们有下面这段 C#代码：

```csharp
// csharp
var a = 11;
var b = a + 9;
```

我们有两个赋值语句（并且有先后依赖），如何用表达式的方式来重写？

```csharp
// csharp
// we build this helper function for next use.
public int Eval(int binding, Func<int, int> continues) {
    return contineues(binding);
}

// then, below sample is totally one expression.
Eval(11, a =>
    //now a is binding to 11
    Eval(a + 9, b => b
        // now, b is binding to a + 9,
        // which is evaluate to 11 + 9
    ));
```

这里使用了函数闭包，我们会在接下来的柯里化部分继续谈到。通过使用 continues（延续）技术以及闭包，我们成功的将赋值语句变了函数式的表达式，这也是 F#中`let`的基本工作方式。

## 高阶函数

`一类函数`特性使得高阶函数成为可能。何为高阶函数？高阶函数(higher-order function)就是指能函数自身能够接受函数，并可以返回函数的一种函数。我们来看下面两个例子：

```csharp
// C#
var filteredData = Products.Where(p => p.Price > 10.0);
```

```javascript
// javascript
var timer = setInterval(function () {
  console.log("hello world.");
}, 1000);
```

C#中的`Where`接受了一个匿名函数（Lambda 表达式），所以它是一个高阶函数，javascript 的`SetInterval`函数接受一个匿名的回调函数，因而也是高阶的。

我们用一个更加有表现力的例子来说明高阶函数可以提供的强大能力：

```fsharp
// fsharp
let addBy value = fun n -> n + value
let add10 = addBy 10
let add20 = addBy 20

let result11 = add10 1
let result21 = add20 1
```

`addBy`函数接受一个值 value，并返回一个匿名函数，该匿名函数对参数 n 和闭包值 value 相加后返回结果。也就是说，`addBy`函数通过传入的参数，返回了一个经过定制的函数。

高阶函数使函数定制变的容易，它可以隐藏具体的执行细节，将可定制的部分（或行为）抽象出来并传给某个高阶函数使用。

> 是的，这听起来很像是 OO 设计模式中的模板方法，在 FP 中并没有模板方法的概念，使用高阶函数就可以达到目的了。

在下节的柯里化部分将会看到，这种定制函数的能力内建在很多 FP 语言中，Haskell、F#中都有提供。

在 FP 中最常用的就是`map`、`filter`、`fold`了，我们通过检查在 F#中它们的签名就可以推测它们的用途：

```
map:    ('a -> 'b) -> 'a list -> 'b list
filter: ('a -> bool) -> 'a list -> 'a list
fold:   ('a -> 'b -> 'a) -> 'a -> 'b list -> 'a
```

`map`通过对列表中的每个元素执行参数函数，得到相应的结果，是一种映射。C#对应的操作为`Select`。
`filter`通过对列表中的每个元素执行参数函数，将结果为`true`的元素返回，是一种过滤。C#对应的操作为`Where`。
`fold`相对复杂一些，我们可以理解为一种带累加器的化简函数。C#对应的操作为`Aggregate`。

之前我们提到过，泛型本身可以看做是某种类型的包装，所以如果我们面对一个`'T list`，那么我们可以说这是一个`'T`类型的**包装**，注意此处并没有说它是个范型列表。于是乎，我们对`map`有了一种更加高层次的理解，我们可以尝试一种新的签名：`('a -> 'b) -> M<'a> -> M<'b>`，这就是说，`map`将拆开包装，对包装内类型进行转换产生某种新的类型，然后再以同样的包装将其重新打包。

`map`也叫普通投影，请记住这个签名，我们在最后的延续一节将提出一个新的术语叫**平展投影**，到时候还会来对比`map`。

如果我们对两个甚至是三个包装类型的值进行投影呢？我们会猜想它的签名可能是这样：

- lift2: `('a -> 'b -> 'c) -> M<'a> -> M<'b> -> M<'c>`
- lift3: `('a -> 'b -> 'c -> 'd) -> M<'a> -> M<'b> -> M<'c> -> M<'d>`

其实这便是 FP 中为人们广泛熟知的“提升”，它甚至可以称作是一种函数式设计模式。提升允许将一个对值进行处理的函数转换为一个在不同设置中完成相同任务的函数。

## 柯里化和部分函数应用

> 在计算机科学中，柯里化（Currying）是把接受多个参数的函数变换成接受一个单一参数（最初函数的第一个参数）的函数，并且返回接受余下的参数且返回结果的新函数的技术。

这段定义有些拗口，我们借助前面的一个例子，并通过 javascript 来解释一下：

```javascript
// javascript
function addBy(value) {
  return function (n) {
    return n + value;
  };
}

var add10 = addBy(10);
var result11 = add10(1);
```

javascript 版本完全是 F#版本的复刻，如果我们想换个方式来使用它呢？

```
var result11 = addBy(10, 1);
```

这明显是不可以的（并不是说不能调用，而是说结果并非所期望的），因为`addBy`函数只接收一个参数。但是柯里化要求我们函数只能接受一个参数，该如何处理呢？

```javascript
var result11 = addBy(10)(1);
//             ~~~~~~~~~    return an anonymous fn(anonymousFn, e.g)
```

如此就可以了，`addBy(10)`将被正常调用没有问题，返回的匿名函数又立即被调用`anonymousFn(1)`，结果正是我们所期望的。

假如 javascript 在调用函数时可以像 Ruby 和 F#那样省略括号呢？我们会得到`addBy 10 1`，这和真实的多参数函数调用就更像了。在`addBy`函数内部，返回匿名函数时带出了`value`的值，这是一个典型的闭包应用。在`addBy`调用后，`value`值将在外部作用域中不可见，而在返回的匿名函数内部，`value`值仍然是可以采集到的。

> 闭包（Closure）是词法闭包（Lexical Closure）或函数闭包（function closures）的简称，可参见[wikipedia](<https://en.wikipedia.org/wiki/Closure_(computer_programming)>)上的详细解释。

如此看来，是不是所有的多参数函数都能被柯里化呢？我们假想一个这样的例子：

```
function fakeAddFn(n1) {
    return function(n2) {
        return function(n3) {
            return function(n4) {
                return n1 + n2 + n3 + n4;
            };
        };
    };
}

var result = fakeAddFn(1)(2)(3)(4);
//           ~~~~~~~~~~~~           now is function(n2)
//                       ~~~        now is function(n3)
//                          ~~~     now is function(n4)
//                             ~~~  return n1 + n2 + n3 + n4
```

但是这样又显得非常麻烦并且经常会出现智商不够用的情况，如果语言能够内建支持 currying，那么情况将乐观许多，例如 F#可以这样做：

```fsharp
let fakeAddFn n1 n2 n3 n4 = n1 + n2 + n3 + n4
```

编译器将自动进行柯里化，完全展开形式如下：

```fsharp
let fakeAddFn n1 = fun n2 -> fun n3 -> fun n4 -> n1 + n2 + n3 + n4
```

并且 F#调用函数时可以省略括号，所以对`fakeAddFn`的调用看上去就像是对多参数函数的调用：`let result = fakeAddFn 1 2 3 4`。到这里你也许会问，currying 到底有什么用呢？答案是：部分函数应用。

由于编译器自动进行 currying，所以每一个函数本身是可以部分调用的，举个例子，F#中的`+`运算符其实是一个函数，定义如下：

```fsharp
let (+) a b = a + b
```

利用前面的知识我们知道它的完全形式是这样：

```
let (+) a = fun b -> a + b
```

所以我们自然可以编写一个表达式只给`+`运算符一个参数，这样返回的结果是另一个接受一个参数的函数，之后，再传入剩余一个参数。

```fsharp
let add10partial = (+) 10
let result = add10partial 1
```

同时，由于`add10partial`函数的签名是`int -> int`，所以可以直接用于`List.map`函数，如下：

```fsharp
let add10partial = (+) 10
let result = someIntList |> List.map add10partial

// upon expression equals below
// let result = List.map add10partial someIntList

// or, more magic, make List.map partially:
let mapper = (+) 10 |> List.map
let sameResult = someIntList |> mapper
```

> `|>`运算符本身也是一个函数，简单的定义就是`let (|>) p f = f p`，这种类似管道的表达式为 FP 提供了更高级的表达。

我们知道 FP 是以`Alonzo Church`的 lambda 演算为理论基础的，lambda 演算的函数都是接受一个参数，后来`Haskell Curry`提出的 currying 概念为 lambda 演算补充了表示多参数函数的能力。

## 递归及优化

由于 FP 没有可变状态的概念，所以当我们以 OO 的思维来思考时会觉得无从下手，在这个时候，递归就是强有力的武器。

> 其实并不是说现代的 FP 语言没有可变状态，其实几乎所有的 FP 语言都做了一定程度的妥协，诸如 F#构建在.NET 平台之上，那么在与 BCL 提供的类库互操作时避免不了要涉及状态的改变，而且如果全部使用递归的方式来处理可变状态，在性能上也是一个严峻的考验。所以 F#其实提供了可变操作，但是需要明确的使用`mutable`关键字来声明或者使用`引用单元格`。

以一个典型的例子为开始，我们实现一个 Factorial 阶乘函数，如果以命令式的方式来实现是这样的：

```csharp
// csharp
public int Factorial(int n) {
    var result = 1;
    for(int index = 2; index <= n; index++) {
        result = result * index;
    }
    return result;
}
```

这是典型的 how to do，我们开始尝试用递归并且尽可能的用表达式来解决问题：

```csharp
// csharp
public int Factorial(int n) {
    return n <= 1
        ? 1
        : n * Factorial(n - 1);
}
```

这段代码是可以正常工作的，但是如果 n 的值为 10,000 呢？会栈溢出。此时便出现了本节要解决的第二个问题：递归优化。

那么这段递归代码为什么会溢出？我们展开它的调用过程：

```
n               (n-1)       ...      3         2       1  // state
--------------------------------------------------------
n*f(n-1) -> (n-1)*f(n-2) -> ... -> 3*f(2) -> 2*f(1) -> 1  // stack in
                                                       |
n*r      <-  (n-1)*(r-1) <- ... <-   3*2  <-   2*1  <- 1  // stack out
```

简单来说，因为当`n`大于 1 时，每次递归都卡在了`n * _`上，必须等后面的结果返回后，当前的函数调用栈才能返回，久而久之就会爆栈。那可以做点什么呢？如果我们在递归调用的时候不需要做任何工作（例如不去乘以 n），那么就可以从当前的调用栈直接跳到下一次的调用栈上去。这称为尾递归优化。

我们考虑，当前调用时的 n，如果以某种形式直接带到下一次的递归调用中，那么是不是就达到了目的？没错，这就是累加器技术，来尝试一下：

```csharp
private int FactorialHelper(acc, n) {
    return n <= 1
        ? acc
        : FactorialHelper(acc * n, n - 1);
}

public int Factorial(int n) { return FactorialHelper(1, n); }
```

C#毕竟没有 F#那么方便的内嵌函数支持，所以我们声明了一个 Helper 函数用来达到目的，对应的 F#实现如下：

```fsharp
let factorial n =
    let rec helper acc n' =
        if n' <= 1 then acc
        else helper (acc * n') (n' - 1)
    helper 1 n
```

下面的示意表达了我们想达到的效果：

```
init        f(1, n)             // stack in
                |               // stack pop, jump to next
n           f(n, n-1)           // stack in
                |               // stack pop, jump to next
n-1         f(n*(n-1), n-2)     // stack in
                |               // stack pop, jump to next
...         ...                 // stack in
                |               // stack pop, jump to next
3           f((k-2), 2)         // stack in
                |               // stack pop, jump to next
2           f((k-1), 1)         // stack in
                |               // stack pop, jump to next
1           k                   // return result
```

可以看到，调用展开成尾递归的形式，从而避免了栈溢出。尾递归是一项基本的递归优化技术，其中关键的就是对累加器的使用。几乎所有的递归函数都可以优化成尾递归的形式，所以掌握这项技能对编写 FP 程序是有重要的意义的。

假如我们遇到的是一个非常庞大的列表需要处理，例如找到最大数或者列表求和，那么尾递归技术也将会让我们避免在深度的遍历时发生栈溢出的情形。

在前面我们说过`fold`是一种自带累加器的化简函数，那么列表求和以及最大数查找是不是可以直接用`fold`来实现呢？我们来尝试一下。

```fsharp
// fsharp
let sum l = l |> List.fold (+) 0
let times l = l |> List.fold (*) 1

let max l =
    let compare s e = if s > e then s else e
    l |> List.fold compare 0
```

可以看到，`fold`抽取了遍历并化简的核心步骤，仅将需要自定义的部分以参数的形式开放出来。这也是高阶函数组合的威力。

> 还有一个和`fold`很类型的术语叫`reduce`，它和`fold`的唯一区别在于，`fold`的累加器需要一个初始值需要指定，而`reduce`的初始累加器使用列表的第一个元素的值。

## 记忆化

我们知道大多数的 FP 函数是没有副作用的，这意味着以相同的参数调用同一函数将会返回相同的结果，那么如果有一个函数会被调用很多次，为什么不把对应参数的求值结果缓存起来，当参数匹配时直接返回缓存结果呢？这个过程就是记忆化，也是 FP 编程中常用的技巧。

我们以一个简单的加法函数为例：

```fsharp
let add (a, b) = a + b
```

注意这里我们使用了非 currying 化的参数，它是一个元组。接下来我们尝试使用记忆化来缓存结果：

```fsharp
let memoizedAdd =
    let cache = new Dictionary<_, _>()
    fun p ->
        match cache.TryGetValue(p) with
        | true, result -> result
        | _ ->
            let result = add p
            cache.Add(p, result)
            result
```

借助一个字典，将已经求值的结果缓存起来，下次以同样的参数调用时就可以直接从字典中检索出值，避免了重新计算。

我们甚至可以设计一个通用的记忆化函数，用于将任意函数记忆化：

```fsharp
let memorize f =
    let cache = new Dictionary<_, _>()
    fun p ->
        match cache.TryGetValue(p) with
        | true, result -> result
        | _ ->
            let result = f p
            cache.Add(p, result)
            result
```

那么前面的`memorizedAdd`函数可以写为`let memorizedAdd = memorize add`。这也是一个高阶函数应用的好例子。

## 惰性求值

Haskell 是一种纯函数语言，它不允许存在任何的副作用，并且在 Haskell 中，当表达式不必立即求值时是不会主动求值的，换句话说，是延迟计算的。而在大多数主流语言中，计算策略却是即时计算的（eager evaluation），这在某种极端情况下会不经意的浪费计算资源。有没有什么方法能够模拟类似 Haskell 中的延迟计算？

假如我们需要将表达式`n % 2 == 0 ? "right" : "wrong"`绑定到标识（即变量名）`isEven`上，例如`var isEven = n % 2 == 0 ? "right" : "wrong";`，那么整个表达式是立即求值的，但是`isEven`可能在某种状况下不会被使用，有没有什么办法能在我们确定需要`isEven`时再计算表达式的值呢？

假如我们将`isEven`绑定到某种结构上，这个结构知道如何求值，并且是按需求值的，那么我们的目的就达到了。

```csharp
// csharp
var isEven = new Lazy<string> (() => n % 2 == 0 ? "right" : "wrong");
```

```
// fsharp
let isEven = lazy (if n % 2 = 0 then "right" else "wrong")
```

当使用`isEven`时，C#可以直接使用`isEven.Value`来即时求值并返回结果，而 F#的使用方式也是一样的`isEven.Value`。

还有一种更加通用的方式来实现惰性求值，就是通过函数，函数表达了某种可以得到值的方式，但是需要调用才能得到，这和惰性求值的思想不谋而合。我们可以改写上面的例子：

```csharp
// csharp
var isEven = (Func<string>)(() => n % 2 == 0 ? "right" : "wrong");
```

```fsharp
// fsharp
let isEven = fun () -> if n % 2 = 0 then "right" else "wrong"
```

这样，在需要使用`isEven`的值时就是一个简单的函数调用，C#和 F#都是`isEven()`。

## 延续

如果你之前使用过 jQuery，那么在某种程度上已经接触过延续的概念了。
通过 jQuery 发起 ajax 调用其实就是一种延续：

```javascript
$.get("http://test.com/data.json", function (data) {
  // processing.
});
```

ajax 调用成功后会调用匿名回调函数，而此函数表达了我们希望 ajax 调用成功后继续执行的行为，这就是延续。

现在，我们回顾一下，在概述-表达式求值一节，我们为了将两个 C#赋值语句改写成表达式的方式，新增了一个`Eval`函数：

```csharp
// csharp
public int Eval(int binding, Func<int, int> continues) {
    return contineues(binding);
}
```

它也是一种延续，指定了在`binding`求值后继续执行延续的行为，我们将它稍做修改：

```csharp
// csharp
public TOutput binding<TEvalValue, TOutput>(
    TEvalValue evaluation,
    Func<TEvalValue, TOutput> continues) {

    return continues(evaluation());
}
```

```fsharp
// fsharp
let binding v cont = cont v
// binding: 'a -> cont:('a -> 'b) -> 'b
```

于是我们可以模拟`let`的工作方式：

```fsharp
// fsharp
binding 11 (fun a -> printfn "%d" a)
```

那么延续这种技术在实践中有什么用途呢？你可以说它就是个回调函数，这没有问题。深层次的理解在于，它延后了某种**行为**且该行为对上下文有依赖。

我们考虑这样一个场景，假设我们有一颗树需要遍历并求和，例如：

```fsharp
// fsharp
type NumberTree =
    | Leaf of int
    | Node of NumberTree * NumberTree

let rec sumTree tree =
    match tree with
    | Leaf(n)           -> n
    | Node(left, right) -> sumTree(left) + sumTree(right)
```

那么问题来了，我们显然可以发现当树的层级太深时`sumTree`函数会发生栈溢出，我们也自然而然的想到了使用尾递归来优化，但是当我们在尝试做优化时会发现，然并卵。这就是一个无法使用尾递归的场景。

核心的诉求在于，我们希望进行尾递归调用（`sumTree(left)`），但在尾递归调用完成之后，还有需要执行的代码（`sumTree(right)`）。延续为我们提供了一种手段，在函数调用结束后自动调用指定的行为（函数），于是当前正在编写的函数便仅包含一次递归调用了。我们仍然可以将它看作是一种累加器技术，区别在于，之前是累加值，而延续是累加行为。

我们尝试为`sumTree`递归函数加上延续功能：

```fsharp
// fsharp
let rec sumTree tree continues =
    match tree with
    | Leaf(n) -> continues n
    | Node(left, right) ->
        sumTree left (fun leftSum ->
            sumTree right (fun rightSum ->
                continues(leftSum + rightSum)))
```

此时，`sumTree`的签名从`NumberTree -> int`变成了`NumberTree -> (int -> 'a) -> 'a`。`Node(left, right)`分支现在变成了单个函数的调用，所以它是尾递归优化的，每次计算时都会将结束后需要继续执行的行为以函数的方式指定，直到整个递归完成。

使用时，可以以延续的方式来调用`sumTree`函数，也可以像往常一样从返回值获取结果：

```fsharp
// fsharp
// continues way:
sumTree sampleTree (fun result -> printfn "result: %d" result)

// normal way:
let result = sumTree sampleTree (fun r -> r)
```

我们甚至可以从延续的思想逐渐推导出类似`bind`的函数，我们将它与`map`的签名对比：

```
// bind
('a -> M<'b>) -> M<'a> -> M<'b>
// map
('a -> 'b)    -> M<'a> -> M<'b>
```

在高阶函数一节我们说过，`map`叫普通投影，而新的`bind`叫做平展投影，它是一种外层匹配模式，在 C#中对应的操作是`SelectMany`，在 F#中就是`bind`，是一种通用函数。

在前面我们定义了一个`binding`函数，我们稍微调整一下参数顺序，并把它和`bind`对比：

```
// binding:
('a -> 'b)    -> 'a -> 'b
// map:
('a -> 'b)    -> M<'a> -> M<'b>
// bind:
('a -> M<'b>) -> M<'a> -> M<'b>
```

也就是说，如果我们为`'a`加上某种包装，然后在 bind 里再做一些转换，那么我们就可以推导出`bind`函数。

C#的 LINQ 里`SelectMany`对应的就是`from`语句，比如下面：

```csharp
var result = from a in l1
             from b in l2
             from c in l3
             select { a, b }
```

这将转换成一系统嵌套的`SelectMany`调用，而`select`将转换为某种类似于`Return<T>()`的操作。对于 F#来说，类似的代码可以用计算表达式（或者更加具体的序列表达式）：

```
let result = seq {
    let! a = l1
    let! b = l2
    let! c = l3
    return (a, b)
}
```

到这里，似乎差不多该结束了，我们不打算继续深究`bind`，因为再往下走就到了`monad`了。事实上，大家已经看到了`monad`，F#的序列表达式以及 C#中 LINQ 的一部分操作，就是`monad`。

---

希望本文讲述的一些浅显的函数式编程概念可以在实践中对你有所帮助。最重要的是通过对思维的训练，可以从更加抽象的角度思考问题，提取问题最核心的部分以复用，将可变部分提出，从而使问题可组合，并且获得更好的表达性。

有关`monad`，推荐大家看看[Erik Meijer](https://en.wikipedia.org/wiki/Erik_Meijer_%28computer_scientist%29)大大在 Channel9 上的课程[Functional Programming Fundamentals](https://channel9.msdn.com/Series/C9-Lectures-Erik-Meijer-Functional-Programming-Fundamentals)，它同时也是[Rx](https://github.com/Reactive-Extensions)库的作者之一，以及 LINQ 的作者。

（完）
