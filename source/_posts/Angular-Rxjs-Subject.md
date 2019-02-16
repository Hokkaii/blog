---
title: Angular RxJS-Observable-Observer-Subject
date: 2019-2-14 17:21:37
tags:
---
## 观察者模式
观察者模式是JavaScript中一种经典的设计模式。具体实现是：一个目标对象管理所有依赖于它的所有观察者对象，并且在它自身发生变化的时候主动发出通知（前文异步编程方案一文中曾经使用这一设计模式解决异步编程问题）。在JavaScript中这一模式最常见的就是用户事件触发回调函数。

---
<!--more-->
## 迭代器模式
在不需要对象内部显示的情况下，提供一种方法可以顺序的访问聚合对象中的各个元素。
JavaScript中的迭代器模式：

### Es5
```
    function makeIterator(array){
        var nextIndex = 0;
        return {
            next: function(){
                return nextIndex < array.length ?
                    {value: array[nextIndex++], done: false} :
                    {done: true};
            }
        }
    }
```
使用一个函数，这个函数可以返回一函数，这个函数依据父函数的参数（数组）的长度来判断来返回相应的信息。

### Es6
```
    let arr = ['a', 'b', 'c'];
    let iter = arr[Symbol.iterator]();
```
在 ES 6 中我们可以通过 Symbol.iterator 来创建可迭代对象的内部迭代器。

---
## RxJS
RxJS 是基于观察者模式和迭代器模式以函数式编程思维来实现的。RxJS 中含有两个基本概念：Observables 与 Observer。Observables 作为被观察者，是一个值或事件的流集合；而 Observer 则作为观察者，根据 Observables 进行处理。即 **观察者以迭代器模式来订阅发布者主动发布的数据**。

---

## Observable与Observer
Observable 作为被观察者，是一个值或事件的流集合；而 Observer 则作为观察者，根据 Observable进行处理。

最简单的手写一个observable的实现：
### DataSource - 数据源
```
    class DataSource {

        constructor() {
            let i = 0;
            this._id = setInterval(() => this.emit(i++), 200); // 创建定时器
        }
        
        emit(n) {
            const limit = 10;  // 设置数据上限值
            if (this.ondata) {
                this.ondata(n);
            }
            if (n === limit) {
                if (this.oncomplete) {
                    this.oncomplete();
                }
                this.destroy();
            }
        }

        destroy() { // 清除定时器
            clearInterval(this._id);
        }

    }
```
### myObservable
```
    function myObservable(observer) {
        let datasource = new DataSource(); // 创建数据源
        datasource.ondata = (e) => observer.next(e); // 处理数据流
        datasource.onerror = (err) => observer.error(err); // 处理异常
        datasource.oncomplete = () => observer.complete(); // 处理数据流终止
        return () => { // 返回一个函数用于，销毁数据源
            datasource.destroy();
        };
    }
```
### 使用示例
```
    const unsub = myObservable({
        next(x) { console.log(x); },
        error(err) { console.error(err); },
        complete() { console.log('done')}
    });

    // 移除注释，可以测试取消订阅
    // setTimeout(unsub, 500); 
```
上书代码数据源类使用定时器抛出数据，myObservable函数中使用数据源的实例对象；这个实例对象每次抛出数据通过一个observer（包含三个回调函数的对象）处理数据且最后返回一个方法用于销毁数据源，最后的结果是打印1到10十个数字以及"done"。
这就是一Observable抛出数据被订阅者订阅的简单过程：Observable作为一个函数，接受一个 Observer 对象 (包含 next、error、complete 方法的对象) 作为参数，返回一个 unsubscribe 函数，用于取消订阅。observer是一个包含 next、error、complete 方法的对象。

---
## RxJS中的Observable

### Rx.Observable.create

```
    var observable = Rx.Observable
        .create(function(observer) {
            observer.next('Kobe'); // RxJS 4.x 以前的版本用 onNext
            observer.next('bryant');
        });
        
    // 订阅这个 Observable    
    observable.subscribe(function(value) {
        console.log(value);
    });
```
上书代码的执行结果是控制台先后打印'Kobe'与'bryant'。

### Observable - Creation Operator

RxJS中其他的创建Observable的方式：
- create
- of
- from
- fromEvent
- fromPromise
- empty
- never
- throw
- interval
- timer


of命令生成Observable：
```
    var source = Rx.Observable.of('Semlinker', 'Lolo');
    source.subscribe({
        next: function(value) {
            console.log(value);
        },
        complete: function() {
            console.log('complete!');
        },
        error: function(error) {
            console.log(error);
        }
    });
```
......

## Observable vs Promise
做一下JavaScript中的Promise与RxJS中的Observable的比较：
1. 执行情况:promise在创建之时便会执行（因此我们常常使用一个函数返回一个promise）。Observable在创建后并不会立即执行，而是会在调用后执行。
2. 状态：promise只有三个状态，也即pending（准备中）、resolved（成功）、rejected（失败），这也就意味着promise只能返回一个值。Observable能返回多个值，因此它更多的状态。
3. 是否可以被中断：promise是一次完成且只返回一个值，因此它是不可以被中断的。Observable可以返回多个值且订阅可以被中断（RxJS的Observable的subscribe()方法会返回一个unsubscribe()方法用于取消订阅）。
4. 操作符使用情况：Observable支持 map、filter、reduce 等操作符，在数据被订阅之前对数据进行预处理（个人认为实际使用意义不大）。promise不支持任何操作符。

---
## Subject
看一个简单的Observable订阅过程：
```
    const interval$ = Rx.Observable.interval(1000).take(3);
    interval$.subscribe({
        next: value => console.log('Observer A get value: ' + value);
    });
    setTimeout(() => {
        interval$.subscribe({
            next: value => console.log('Observer B get value: ' + value);
        });
    }, 1000);
```

上书代码的结果：
```
    Observer A get value: 0
    Observer A get value: 1
    Observer B get value: 0
    Observer A get value: 2
    Observer B get value: 1
    Observer B get value: 2
```

可以看得出Observable 对象可以被重复订阅而且Observable 对象每次被订阅后，都会重新执行。这是Observable的默认场景，现在希望不要重复订阅，比如第二个订阅者可以得到与第一个订阅者相同状态的返回值。这个时候需要一个特定的Subject:
```
    class Subject {   
        constructor() {
            this.observers = [];
        }
        
        addObserver(observer) { 
            this.observers.push(observer);
        }
        
        next(value) {  
            this.observers.forEach(o => o.next(value));    
        }
    
        error(error){ 
            this.observers.forEach(o => o.error(error));
        }
        
        complete() {
            this.observers.forEach(o => o.complete());
        }
    }
```
使用:
```
    const interval$ = Rx.Observable.interval(1000).take(3);
    let subject = new Subject();

    let observerA = {
        next: value => console.log('Observer A get value: ' + value),
        error: error => console.log('Observer A error: ' + error),
        complete: () => console.log('Observer A complete!')
    };

    var observerB = {
        next: value => console.log('Observer B get value: ' + value),
        error: error => console.log('Observer B error: ' + error),
        complete: () => console.log('Observer B complete!')
    };

    subject.addObserver(observerA); // 添加观察者A
    interval$.subscribe(subject); // 订阅interval$对象
    setTimeout(() => {
        subject.addObserver(observerB); // 添加观察者B
    }, 1000);
```
结果：
```
    Observer A get value: 0
    Observer A get value: 1
    Observer B get value: 1
    Observer A get value: 2
    Observer B get value: 2
    Observer A complete!
    Observer B complete!
```
这个时候达到了预想的效果。而且可以得到一个结论：subject既是一个Observer对象又是一个Observable 对象。当有新消息时，Subject 会对内部的 observers 列表进行组播 (multicast)。

---
## RxJS Subject
RxJS Subject源码：
```
    export declare class Subject<T> extends Observable<T> implements SubscriptionLike {
        observers: Observer<T>[];
        closed: boolean;
        isStopped: boolean;
        hasError: boolean;
        thrownError: any;
        constructor();
        /**@nocollapse */
        static create: Function;
        lift<R>(operator: Operator<T, R>): Observable<R>;
        next(value?: T): void;
        error(err: any): void;
        complete(): void;
        unsubscribe(): void;
        /** @deprecated This is an internal implementation detail, do not use. */
        _trySubscribe(subscriber: Subscriber<T>): TeardownLogic;
        /** @deprecated This is an internal implementation detail, do not use. */
        _subscribe(subscriber: Subscriber<T>): Subscription;
        asObservable(): Observable<T>;
    }
```
可以看到Subject类是Observable类的继承类同时还实现了SubscriptionLike接口。
在Angular 2 中RxJS Subject 的最直接应用就是任意组件间的传值：

在服务文件message.service.ts中：
```
    import { Injectable } from '@angular/core';
    import {Observable} from 'rxjs/Observable';
    import { Subject } from 'rxjs/Subject';

    @Injectable()
    export class MessageService {
        private subject = new Subject<any>();

        sendMessage(message: string) {
            this.subject.next({ text: message });
        }

        clearMessage() {
            this.subject.next();
        }

        getMessage(): Observable<any> {
            return this.subject.asObservable();
        }
    }
```
在home.component.ts中：
```
    import { Component } from '@angular/core';

    import { MessageService } from '../_services/index';

    @Component({
        moduleId: module.id,
        templateUrl: 'home.component.html'
    })

    export class HomeComponent {
        constructor(private messageService: MessageService) {}
        
        sendMessage(): void { // 发送消息
            this.messageService.sendMessage('Message from Home Component to App Component!');
        }

        clearMessage(): void { // 清除消息
            this.messageService.clearMessage();
        }
    }
```
在app.component.ts中：
```
    import { Component, OnDestroy } from '@angular/core';
    import { Subscription } from 'rxjs/Subscription';

    import { MessageService } from './_services/index';

    @Component({
        moduleId: module.id,
        selector: 'app',
        templateUrl: 'app.component.html'
    })

    export class AppComponent implements OnDestroy {
        message: any;
        subscription: Subscription;

        constructor(private messageService: MessageService) {
            this.subscription = this.messageService.getMessage()
                .subscribe(message => { this.message = message; });
        }

        ngOnDestroy() {
            this.subscription.unsubscribe();
        }
    }
```
上书代码实现了从home组件传值至app入口组件。在传值时，subject作为Observable对象使用next方法发送信息；在接受值的时候在getMessage函数中通过asObservable()方法变作为Observer对象接收信息再作处理。这种传值方式不限于父子组件或者是路由相关组件，类似于状态管理。
