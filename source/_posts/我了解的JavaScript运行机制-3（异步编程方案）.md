---
title: 我了解的JavaScript运行机制-3（异步编程方案）
date: 2018-12-13 09:38:33
tags:
---
由于JavaScript是一门单线程语言，因此异步编程是必要的。本文希望总结个人理解的多种异步编程方案。
<!--more-->
## 回调函数
首先是回调函数：执行异步任务的方法会提供几个参数，通常最后一个参数会是一个类型为函数的参数，我们称这个函数为回调函数。

```
function async(arg1,arg2,arg3,callback) {
    setTimeout(() => {
        var a  = arg1+arg2+arg3;
        callback(a);
    }, 2000);
}   
async(1,2,3,back);
function back(e){
    console.log(e)
}
```

这是一个简单的包含回调函数执行异步任务的方法。通过setTimeout创建了简单的异步任务，传送back函数来拿到异步任务返回的值并进行一些操作。
复杂的情况：

```
function async(arg1,arg2,arg3,callback) {
    setTimeout(() => {
        var a  = arg1+arg2+arg3;
        callback(a);
    }, 2000);
}

function async1(arg1,arg2,arg3,callback) {
    setTimeout(() => {
        var a  = arg1+arg2+arg3;
        callback(a);
    }, 2000);
}

function async2(arg1,arg2,arg3,callback) {
    setTimeout(() => {
        var a  = arg1+arg2+arg3;
        callback(a);
    }, 2000);
}  

async(1,2,3,function back(e){
    async1(e,3,4,function back2(r){
        async2(r,5,6,function back(t){
            console.log(t)
        }); 
    });
});
```

如代码所示，我们希望在得到async函数的结果后调用async1，再在async1函数结果基础上调用async2，这样就形成了多个回调函数的多层嵌套。比较成熟的异步方法一般都提供了错误处理方式，因此**回调函数本身并没有问题**，但是多层回调函数看起来肯定是臃肿的，复杂的和不方便阅读的。这也就是"回调地狱"。

---
## 事件监听
>事件:JavaScript 使我们有能力创建动态页面。事件是可以被 JavaScript 侦测到的行为(w3c)。
>任务的执行不取决于代码的顺序，而取决于某个事件是否发生。

以原生Ajax请求为例：

```
function ajax_method(url,data,method,success) {
    var ajax = new XMLHttpRequest();
    if (method=='get') {
        if (data) {
            url+='?';
            url+=data;
        } else {
        }
        ajax.open(method,url);
        ajax.send();
    } else {
        ajax.open(method,url);
        ajax.setRequestHeader("Content-type","application/x-www-form-urlencoded");
        if (data) {
           
            ajax.send(data);
        }else{
            ajax.send();
        }
    }
    ajax.onreadystatechange = function () {
        // 在事件中 获取数据 并修改界面显示
        if (ajax.readyState==4&&ajax.status==200) {
            success(ajax.responseText);
        }
    }

}
```
我们对ajax对象添加了onreadystatechange事件监听，如果数据请求成功（或失败），相应的函数就会被调用。
这一方法在jQuery编程中更容易实现：
```
f1.on('done', f2);
function f1(){
　　setTimeout(function () {
        const value = 1;
　　　　f1.trigger('done');
　　}, 1000);
}
function f2(){
    console.log('执行于done事件之后')
}
```

---
## 订阅与发布

订阅发布模式也即观察者模式。
首先强调观察者模式与事件监听模式的联系：JavaScript提供了若干的事件句柄，常见的有click，dblclick，mouseenter等等，使用addEventListener，on等方法为对象添加监听器，这样监听器可以在事件被触发的时候去执行绑定的函数。在jq中，利用trigger方法可以实现简单的自定义事件。
个人理解：观察者模式省去了事件监听器这个中转点，也就是说如果某个观察者订阅了某个主题，这个主题变更时会直接通知这个观察者（订阅这个主题所有的观察者），然后再执行相应的函数。

```
var Event = (function(){
    var list = {},listen,trigger,remove;

    listen = function(key,fn){ //监听事件函数
        if(!list[key]){
            list[key] = []; //如果事件列表中还没有key值命名空间，创建
        }
        list[key].push(fn); //将回调函数推入对象的“键”对应的“值”回调数组
    };

    trigger = function(){ //触发事件函数
        var key = Array.prototype.shift.call(arguments); //第一个参数指定“键”
        msg = list[key];
        if(!msg || msg.length === 0){
            return false; //如果回调数组不存在或为空则返回false
        }
        for(var i = 0; i < msg.length; i++){
            msg[i].apply(this, arguments); //循环回调数组执行回调函数
        }
    };

    remove = function(key, fn){ //移除事件函数
        var msg = list[key];
        if(!msg){
            return false; //事件不存在直接返回false
        }
        if(!fn){
            delete list[key]; //如果没有后续参数，则删除整个回调数组
        }else{
            for(var i = 0; i < msg.length; i++){
                if(fn === msg[i]){
                    msg.splice(i, 1); //删除特定回调数组中的回调函数
                }
            }
        }
    };
    return {listen: listen, trigger: trigger, remove: remove}
})();

var fn = function(data){
    console.log(data + '的推送消息：xxxxxx......');
}

Event.listen('a', fn);

function async(){
    setTimeout(()=>{
        Event.trigger('a', '2016.11.26');
    },2000)
}

async();

// Event.remove('a', fn);
```

这样就是一个简单的利用观察者模式实现的异步编程。

---
## promise
promise是解决异步问题一种很好的方式，也是现在很常用的一种方式。promise汉译为"承诺"，他的调用方法then意为"然后"。**在承诺生效之后可以进行操作。**
简单的promise如下所示：

```
const promise = new Promise(function(resolve, reject) {
    ...
    ...
  if (/* 异步操作成功 */){
    resolve(value);
  } else {
    reject(error);
  }
});
promise.then(value=> {
  // success
}, error=> {
  // failure
});
```
如果有多层嵌套需求：
```
function async1(num) {
    returnnew Promise(function(resolve, reject) {
        ...
    if (/* 异步操作成功 */){
        resolve(post);
    } else {
        reject(error);
    }
    });
}

async1().then(res=> {
    return new Promise(function(resolve, reject){
        resolve(res+1);
    })
}).then(res=>{
    res...
});
```

Promise类的实例化对象在调用then方法后返回的仍然是一个promise对象，因此仍然可以继续"then"，这样就得到了一个"then链"，这种结构就可以帮助我们实现前述的回调函数的多层嵌套的需求。这种使用promise异步编程的方式从条理和代码外观来看都要优于回调函数，我们使用promise一定程度上解决掉了回调地狱问题。
promise将嵌套改为链式。
可以看出的是promise的编程方式仍然存在缺点：首先是过多的then会像回调函数那样臃肿。

---
## async/await
async函数与promise对象实质上没有区别。async函数的return相当于promise对象的resolve；在promise对象内部可以使用reject抛出一个错误（报错等同于promise变为rejected），也可以使用try,catch与throw语句在promise中抛出错误，这几者并无区别。在async函数中也是如此，虽然无法使用reject语句，但是可以使用throw语句抛出错误，这就等同于自身所形成的promise状态变为rejected。

```
function async1(num) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(num)
        }, 2000);
    } )
}

function async2(num) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(num*2)
        }, 2000);
    } )
}

async function get(){
    const value = await async1(1);
    const value2 = await async2(value);
    console.log(value, value2)
}
get()
```

毫无疑问的是，使用async/await来代替promise进行多次回调是一种很好的方式：首先多层嵌套需求不再依赖于then链，我们可以使用更为简洁的await来梳理这些事件调用的顺序；在async函数内部使用await等待promise，实现有依赖关系的嵌套渐进的异步任务。缺点也很明显：在使用async与await的时候必须要依赖于能返回promise对象的函数,虽然async函数也能构建一个promise对象，但是在一个函数中return语句相比于resolve语句受限太大，我们往往无法达到预期的目的:

```
function async1(num) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(num)
        }, 2000);
    } )
}
async function async2(num) {
    setTimeout(() => {
        return num
    }, 2000);
    // return
}

async function get(){
    const value = await async1(1);
    const value2 = await async2(value);
    console.log(value, value2)
}
get()
```

显然这是错误的

---
## Generator&Thunk
简单的概况generetor函数：
```
function* A() {
  yield 'hello';
  yield 'world';
  return 'ending';
}

var a = A();
```
在function标识符与函数名之间添加<font color=red>*</font>，这个函数就是一个Generator函数。Javascript通常的函数在调用后便会执行，执行至<font color=red>return</font>处便结束，要特别指出的是，如果没有报错，这个过程是不间歇的，不停止的。Generator函数函数包含了多个状态，在Generator函数的调用结果上调用next方法可以访问Generator函数的各个产出（yield）的状态，得到一个形如<font color=red>{ value: 'hello', done: false }</font>的对象；每一次的next调用都会使得Generator函数的状态跳转至下一个yield语句之前。
一个异步任务完成后执行另一个异步任务：
```
 function *gen () {
    var result = yield new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve('result')
        }, 2000);
    });
    console.log('异步结果', result);  
}
var g = gen();
var result = g.next();
result.value.then((data)=> {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(data+'_again')
        }, 2000);
    });
}).then(function(data){
    g.next(data);
});
```
虽然这可以实现跟<font color=red>await</font>类似的效果，即不再需要复杂的嵌套或者链式结构就可以实现回调；但是，这种编程方式仍然存在缺点，真正实现并调用的时候前者仍然很复杂。

---

Thunk函数：JavaScript的求值策略是"传值调用"，也就是说，如果调用一个函数需要传参，传入的参数是一个表达式，JavaScript会先对表达式进行计算，然后再调用函数，这被叫做"传值调用"。通常异步任务函数的调用需要传入多个参数（最后一个是回调函数）。通常来说，除回调函数外，其他几个参数都是固定的，我们仅仅需要回调函数来达到不同的变成目的。<font color=red>Thunk</font>函数可以为此而生，这跟JavaScript中的"科里化"很类似。Thunk函数并不常用，但是它与Generator函数结合可以让后者自动执行，实现比较好的异步编程。
```
function *genFun(){
    let result = yield function(fun){
        setTimeout(() => {
            fun({result:'data'});
        }, 2000);
    }
    let result_1 = yield function(fun){
        setTimeout(() => {
            fun({result:'data_1'});
        }, 2000);
    }
    let result_2 = yield function(fun){
        setTimeout(() => {
            fun({result:'data_2'});
        }, 2000);
    }
    console.log('异步操作的结果是:', result, result_1, result_2);
}

// 下边的函数可以把Generator函数转化为Thunk函数
function run(genFun){
    this.next = function(data){
        let result = genFun.next(data);
        // console.log('result value is', data, result);
        if (result.done) {
            return
        } else {
            result.value(this.next);
        }
    };
    next();
}   

var gen = genFun();
run(gen);
```
执行过程分析：Generator函数genFun有4个状态（加上return），前三个状态对应的是三个函数，这三个函数会在<font color=red>2s</font>后执行函数（参数）fun。run函数负责genFun函数的自动化进行，run函数执行后，next变量被赋值为一个函数，执行后genFun被推动至第一个状态，并使用next方法给上一个状态赋值。然后判断这并不是最后一个状态，由于第一个状态值value是一个函数，以next函数为参数调用这个函数（如果是最后一个状态便结束），以此推进状态。最后就可以得到最后的异步结果