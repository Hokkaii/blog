---
title: 我了解的JavaScript运行机制-2（异步与事件循环）
date: 2018-12-08 13:34:22
tags:
---
## 提要
这里只谈JavaScript在浏览器环境中的事件循环（Event Loop），不涉及Nodejs环境的情况。
首先，JavaScript是一门单线程语言。单线程，顾名思义，就意味着代码的执行是由上而下顺序执行的，也就是说只能是一个任务执行完毕之后，随后的任务才可以接着执行。考虑到Javascript是一门浏览器脚本语言，因此前者只能是单线程的。伴随着单线程接踵而至的就是堵塞问题，因为难免有一些任务执行速度比较慢，最常见的Ajax请求便是如此。考虑到用户体验，我们不能允许长时间的页面加载，异步可以很好的解决这个问题，事件循环是异步的底层原理。

---
<!--more-->
## 事件循环
首先明确几个概念：
执行栈：用栈的概念去思考这个问题（压入，弹出）：任务进入栈，执行完被毕弹出栈，这个任务的执行流程就结束了。
主线程：所有的同步任务的执行的过程，同步任务执行时遵照上述进栈出栈流程。
任务队列：当有异步任务就绪（例如Ajax数据请求就绪），它的回调函数便会被放进任务队列中。待所有的同步任务执行结束，执行栈这个时候就是空的，这个时候执行栈开始从任务队列中读取任务并且执行。很明显，任务队列中的异步回调函数排列顺序是由异步任务就绪的先后顺序决定的，也即率先就绪率先被执行。

当执行栈为空时，执行栈每从任务队列中读取并执行完一个任务，就是一轮事件循环。

---
## setTimeout
谈一下个人对setTimeout的认知：最初，在不了解事件循环以及异步的时候，只认为setTimeout是用于“延迟一段时间执行一段代码”；而后在接触异步后才明白setTimeout是一个典型的异步函数，使用setTimeout可以立即创建一个异步任务；再之后更加深入了解事件循环后，就对setTimeout有了更深理解：
```
    async function async1() {
        console.log('async1 start');
        await async2();
        console.log('async1 end');

    }
    async  function async2() {
        console.log( 'async2');
    }
    console.log('script start');
    setTimeout(function () {
        console.log('settimeout');
    },0);
    async1();
    new Promise(function (resolve) {
        console.log('promise1');
        resolve();
    }).then(function () {
        console.log('promise2');
    });
    console.log('script end');

```
首先，任务队列中的异步任务的回调函数执行顺序由它们任务的就绪时间点决决定。Javascript中的常见的异步任务构建方法除上述的setTimeout外，还有setInterval，promise，async等（angular相关的rxjs的observe）。如上所示，我们假设这些异步回调消耗额外的时间就可以就绪，那么异步任务的执行顺序是按代码执行顺序还是另有文章？
```
script start
async1 start
async2
promise1
script end
async1 end
promise2
settimeout

```
开始分析：
首先是同步任务：毫无疑问，首先打印'script start'。然后是async1函数执行，这是一个异步函数，但是异步函数的执行过程仍属于同步任务范围，所以打印'async1 start'。在异步函数async1执行过程中，'await async2()'意味着要等待async2的return（类似于promise的then），毫无疑问，async2被调用了，因此打印async2。继续主线程，创建promise的过程打印'promise1'。紧接着是同步任务最后的'script end'。
然后是异步任务：首先要指出的是：async的return在本质上无异于promise的resolve。因此就只要讨论promise的then（async的await）与setTimeout(fn,0)的先后顺序即可得出结论。从结果来看很明显setTimeout在then与await之后。因此可以得出**表面的结论**：在无额外消耗的情况下，setTimeout(fn,0)的执行顺序在promise回调之后。

setTimeouot的真正执行时间分析：

在阮老师的文章中这样描述这个过程：
> 总之，setTimeout(fn,0)的含义是，指定某个任务在主线程最早可得的空闲时间执行，也就是说，尽可能早得执行。它在"任务队列"的尾部添加一个事件，因此要等到同步任务和"任务队列"现有的事件都处理完，才会得到执行。
HTML5标准规定了setTimeout()的第二个参数的最小值（最短间隔），不得低于4毫秒，如果低于这个值，就会自动增加。在此之前，老版本的浏览器都将最短间隔设为10毫秒。另外，对于那些DOM的变动（尤其是涉及页面重新渲染的部分），通常不会立即执行，而是每16毫秒执行一次。这时使用requestAnimationFrame()的效果要好于setTimeout()。
需要注意的是，setTimeout()只是将事件插入了"任务队列"，必须等到当前代码（执行栈）执行完，主线程才会去执行它指定的回调函数。要是当前代码耗时很长，有可能要等很久，所以并没有办法保证，回调函数一定会在setTimeout()指定的时间执行。

---
文中指出：**它（setTimeout(fn,0)）在"任务队列"的尾部添加一个事件，因此要等到同步任务和"任务队列"现有的事件都处理完，才会得到执行。**在上方代码中，主线程执行到setTimeout(fn,0)时，事件队列中可能只有异步函数async2()已经就绪，其他的异步任务都没还有就绪。我们可以试着用4ms来解释它，也就是说setTimeout(fn,0)的异步任务就绪至少需要4ms，在这段时间里，其他异步任务已经就绪，因此它在事件队列最后，这几乎可以解释一切。

---
## 事件队列分类
>宏任务队列：script（全局任务）, setTimeout, setInterval, setImmediate, I/O, UI rendering, 网络请求Ajax
>微任务队列：process.nextTick, Promise, Object.observer, MutationObserver
1. 取一个宏任务来执行。执行完毕后，下一步。
2. 取一个微任务来执行，执行完毕后，再取一个微任务来执行。直到微任务队列为空，执行下一步。
3. 更新UI渲染。
> Event Loop 会无限循环执行上面3步，这就是Event Loop的主要控制逻辑。其中，第3步（更新UI渲染）会根据浏览器的逻辑，决定要不要马上执行更新。毕竟更新UI成本大，所以，一般都会比较长的时间间隔，执行一次更新。
![Javascript事件循环](https://image-static.segmentfault.com/402/025/4020255170-59bc9e1671029_articlex "宏任务与微任务")
由于setTimeout属宏任务，因此它一定在promise回调之后。

[事件队列分类引用链接点我](https://segmentfault.com/a/1190000011198232)
仅个人理解与引用，如有错误，日后完善。