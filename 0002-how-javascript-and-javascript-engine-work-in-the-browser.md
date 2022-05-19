---
title: 'JS 与 JS 引擎是如何在浏览器中工作的'
date: '2022-05-18'
tags: 'JavaScript'
---

[原文在此](https://medium.com/jspoint/how-javascript-works-in-browser-and-node-ab7d0d09ac2f)

本文将讨论 `JS` 的调用栈、事件循环、任务队列等。

## JS 概览

JS 是`解释型`语言，同时也是动态类型语言。缺乏类型系统导致`JS`运行缓慢。



## `JS` 引擎剖析

[`ECMAScript`](https://en.wikipedia.org/wiki/ECMAScript) 规范规定了浏览器如何实现 `JS`，但并没有规定 `JS` 在浏览器中如何运行。每个浏览器都提供了 `JS` 引擎来运行 `JS` 代码。


![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0002/NetscapeSpiderMonkeyJavaScriptengine.png)

一个基础的 `JS` 引擎包含基线编译器，用来将 `JS` 代码编译成字节码。基线编译器为了尽快加快编译过程，所以编译出来的代码就不是最优的，因此运行起来就比较慢。


![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0002/2010-v8-js-engine.png)

`2020` 版的 `V8` 引擎使用了 `full-codegen` 作为基线编译器。程序运行时，`crankshaft` 编译器介入并优化代码。优化后的代码运行速度更快。这个 `JIT` 的过程伴随着 `CPU` 和内存的开销增大。

![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0002/2017-v8-js-engine.png)

`2017` 版的 `V8` 改进很多，具体看[这个链接](https://alligator.io/js/v8-engine/)。

## JS 在运行时的样子

`JS` 的运行时是单线程的。因此任何长时间运行的代码都会阻塞后续要执行的任何操作。如图所示：

![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0002/chrome-unresponsive.png)

当你用浏览器打开一个网页，`JS`单线程运行。这个线程负责处理一切，如滚动页面、打印页面内容、监听 `DOM` 事件（用户点击按钮）等。

如果 `JS` 执行被阻塞时，这些事情都将停止运行，也就是说浏览器将卡住不动，直到阻塞任务运行结束。

现代的浏览器为每个标签页或每个域名分分配了单独的线程。如果在两个便签页开了不同域名的网页，其中一个标签页卡住了，并不会影响另外便签页的运行。但如果两个便签页开的是同一个网站，其中一个便签页卡住将导致同域名的另外便签页一并卡住。因为 `Chrome` 采用了每个域名单进程的策略，单进程内使用同一个  `JS` 执行线程。

举例：

```js
function baz() {
  console.log( 'Hello from baz' );
}

function bar() {
  baz();
}

function foo() {
  bar();
}

foo();
```

![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0002/js-runtime.png)

`JS` 运行时有一个堆和一个栈。堆中乱序存储数据。堆由运行时管理，由垃圾收集器清理。想了解更多堆的信息，见此[链接](https://hashnode.com/post/does-javascript-use-stack-or-heap-for-memory-allocation-or-both-cj5jl90xl01nh1twuv8ug0bjk)

再来看栈。栈是后进先出的数据结构，存储了程序中当前函数的执行上下文。只有函数返回了，它才会出栈。一次出栈一个函数，并继续执行下一个函数。

![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0002/stack-frames.gif)

栈中的每一个条目叫栈帧，其中包含了函数调用的信息，如：参数、局部变量、返回地址等。

![调用栈](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0002/chrome-devtool-snippet.png)

`JS`的执行教程叫主进程。



![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0002/browser-under-the-hood.png)

`JS`运行时还包含事件循环和回调队列。回调队列也叫消息队列或任务队列。

除了 `JS` 引擎，浏览器还包含一堆各种各样的功能，如：发送 `HTTP` 请求、监听 `DOM` 事件、`setTimeout` 或 `setInterval` 延迟执行、缓存、数据库存储等。

试想一下，如果这么多任务都跑在 `JS` 线程中的话，体验将无比糟糕。因此像 `HTTP` 请求这类任务，将由浏览器开启单独线程运行，这个过程对 `JS` 透明无感。

浏览器将使用 `C` 或 `C++` 之类的低级语言来实现这个功能。比如，`fetch` 就是这样的。这类 `API` 统称为 `Web APIs`，因为它们不属于 `JS` 规范。

这些 `Web APIs` 都是异步的，调用是需要我们提供回调函数。这些回调函数将于异步操作返回数据后在主进程中执行。

函数返回时，它将出栈并执行下一个栈条目。与此同时，`Web API` 在背后做自己的工作并记住回调函数。一旦此 `Web API` 工作完成，执行结果将被转交给回调函数，并将它俩推入回调队列。

时间循环的唯一任务就是查看回调队列，一旦队列中有回调在等待，就将此回调入栈。一旦栈变空，事件循环将一次推送一个回调入栈。随后，栈中的函数会执行。

以 `setTimeout` 为例，当栈为空时，`setTimeout` 将立即执行。

```js
function printHello() {
    console.log('Hello from baz');
}

function baz() {
    setTimeout(printHello, 3000);
}

function bar() {
    baz();
}

function foo() {
    bar();
}

foo();
```



![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0002/animated-settimeout.gif)

事件循环和回调队列不是 `JS` 引擎的组成部分，而是浏览器或 `Node.js` 提供的运行时。事件循环使用了 `JS` 引擎的 `API` 与之通信，并为之提供了回调函数以供执行。

除了事件循环和回调队列，现代 `JS` 引擎还提供了 微任务队列来执行期约。想了解更多，参见此[链接](https://itnext.io/javascript-promises-and-async-await-as-fast-as-possible-d7c8c8ff0abc)