# Observable-Promise-Async-In-different-Way

## Observable, Promise 和 Async await 三种不同的异步方式之我见 基于Angular

### 前言

> Angular作为一个前端框架之所以有劝退效果的原因之一是 Angular坚持使用 Observer/Observable 设计模式, 随时随地异步返回的都是Rxjs的Observable对象而不是开发者所熟悉的Promise

> 作为一个开发者当他试图开始拿起Angular时面对一系列的新概念是非常令人沮丧的,开发者的直觉反应是使用Observables提供的非常方便的toPromise()方法将Observable转换成Promise进行使用, 但是这样是本末倒置的行为, 抛弃了Observable所既存的巨大优势

> 希望通过此文帮助自己巩固Angular的基本

#### 为什么使用Observable而抛弃Promise

> 简而言之有几条理由, 我们之后溯源而来
1. Observables允许取消正在进行的任务(如果不再需要对`服务器的HTTP请求`或`需求其他一些昂贵的异步操作的结果`，则`Observable的订阅允许取消订阅`，而`Promise即使没有通知的需求最终也会调用成功或失败的回调`)
2. Observables允许返回多个物件(之所以使用物件是因为返回的`东西`是不确定类型的)
3. 单个Observables实例允许存在多个订阅者
4. Observables简化了重试的过程
5. Observable类似于Stream允许传递零个或多个事件并为每个事件调用回调
6. Observable可以从事件等其他来源创建
7. 当异步操作完成或失败时，Promise只会处理单个事件
8. Promise使用了try/catch和async/await更易读的代码


> 假设我们有下述的基于Promise的方法

```javascript
doAsyncPromiseThing()
  .then(() => console.log("I'm done!"))
  .catch(() => console.log("Error"));
```

> 转换上述Promise的方法为Observable的方式并不复杂

```javascript
doAsyncObservableThing()
  .subscribe(
     () => console.log("I'm done!"),
     () => console.log("Error")
  )
```

#### 创建基于Angular的Observable的例子

> Angular使用Rx.js Observables而不是promises来处理HTTP请求

> 假设希望构建一个搜索功能，该功能可以在键入时立即显示结果,就像各大搜索引擎一样, 这样的需求一定不陌生，但是这实际上充满了挑战

1. 搜索不希望用户每次按下某个键时都会触发服务器端点搜索，那样会导致服务端充斥着一系列的http请求,事实上对于后端服务器而言希望在用户停止键入时发送搜索请求而不是每次按键时都发送请求

2. 对于持续的搜索不希望使用相同的查询参数重复对后端服务进行请求

3. 能够处理无序的响应,比如当同一时间存在多个请求时必须考虑请求的响应以意想不到的顺序返回的情况(想象一下我们分先后快速发出两个搜索请求,一个查询汽车时间一个查询火车时间,这样在网络中存在两个请求，但是因为搜索服务分片等后端原因或网络延迟原因，查询火车的请求先于查询汽车的请求回来,这就很尴尬了)

##### 一个基于Promise且无法处理上述边际条件的实例 `search-service.ts`

```typescript
import { Injectable } from '@angular/core';
import { URLSearchParams, Jsonp } from '@angular/http';

@Injectable()
export class searchService {
  constructor(private jsonp: Jsonp) {}

  search (term: string) {
    var search = new URLSearchParams()
    search.set('action', 'opensearch');
    search.set('search', term);
    search.set('format', 'json');
    return this.jsonp
                .get('http://searchAPI.com/search', { search })
                .toPromise()
                .then((response) => response.json()[1]);
  }
}
```

> 构造器中注入Jsonp服务，以针对具有给定搜索内容的search API发出GET请求

> 值得注意的是调用了toPromise以便将`Observable<Response>`转换为`Promise <Response>`最终以`Promise<Array<string>>`作为搜索方法的返回类型

> 在组建中调用`search-service`时

```typescript
import {...} from '...';

@Component({
  selector: 'my-app',
  template: `
    <div>
      <h2>Search</h2>
      <input #term type="text" (keyup)="search(term.value)">
      <ul>
        <li *ngFor="let item of items">{{item}}</li>
      </ul>
    </div>
  `
})
export class AppComponent {
  items: Array<string>;

  constructor(private searchService: searchService) {}

  search(term) {
    this.searchService.search(term)
                         .then(items => this.items = items);
  }
}
```

> 在组件中注入了searchService并通过`search`方法将其功能暴露给模板，模板将`keyUp`事件绑定其上

> 将searchService的搜索方法返回的promise结果展开，并将其作为一个简单的字符串数组暴露给模板，以便使用`*ngFor`循环建立列表

> 当使用Promise时并非无法实现之前的某些边际条件只是为了照顾到边际条件会在view层写很多的页面判断逻辑，但是仍旧无法保证返回数据的先后问题

##### 而当你使用Observable时一切都会变得简单起来

> 首先按照期望修改代码实现`不在每次键入的时候`发送搜索请求，而是在`用户停止键入400ms时`才发送搜索请求

> 为此需要取得一个带有用户输入的搜索词的`Observable<string>`，通过利用Angular的formControl指令代替手动绑定`keyUp`事件

```typescript
import {...} from '...';

@Component({
  selector: 'my-app',
  template: `
    <div>
      <h2>Search</h2>
    //   <input #term type="text" (keyup)="search(term.value)">
    <input type="text" [formControl]="term"/>
      <ul>
        <li *ngFor="let item of items">{{item}}</li>
      </ul>
    </div>
  `
})
export class AppComponent {
  items: Array<string>;
  term = new FormControl();
  constructor(private searchService: searchService) {
       this.term.valueChanges
              .debounceTime(400)        // wait for 400ms pause in events
              .distinctUntilChanged()   // ignore if next search term is same as previous
              .subscribe(term => this.searchService.search(term).then(items => this.items = items));
  }
//   search(term) {
//     this.searchService.search(term)
//                          .then(items => this.items = items);
//   }
}
```

> 在代码的背后，`term`属性会自动暴露名为`valueChanges`实际是`Observable<string>`的属性

> 当有了`Observable<string>`后操作用户输入就和直接操作Observable一样,在这调用`debounceTime(400)`和`distinctUntilChanged()`

> 发送相同内容的搜索实际上在浪费后端服务资源,为了达到理想的期待所要做的就是在调用`debounceTime(400)`后立即调用`distinctUntilChanged()`操作符

> 这样,只有当时间过了400ms并且搜索值变化后后Observable才会抛出一个新的值

#### 用一张图来概括一下

![conclusion of Difference](./assets/Difference.png)

#### 再来一个非Http的Observable例子

```typescript
//root component
import {Component, NgModule} from '@angular/core'
import {BrowserModule} from '@angular/platform-browser'
import {Observable} from 'rxjs/Observable';

@Component({
  selector: 'my-app',
  template: `
    <div>
      <h2>Observable Example</h2>
      <ul>
        <li *ngFor="let message of messages">{{message}}</li>
      </ul>
    </div>
  `,
})
export class App {
  constructor() {
    this.initialTime = Date.now();
    this.messages = [];
    this.log = (m) => {
      const dateDifference = Date.now() - this.initialTime;
      this.messages.push(`${dateDifference}: ${m}`);
    };
    this.doAsyncObservableThing = new Observable(observer => {
      observer.next('Hello, observable world!');
      observer.complete();
    });
    this.doAsyncObservableThing.subscribe(
      this.log
    );
  }
}

@NgModule({
  imports: [ BrowserModule ],
  declarations: [ App ],
  bootstrap: [ App ]
})
export class AppModule {}
```

> 这个spa直接运行的结果很直白，会直接展示出`Hello, observable world!`,因为Observable是立刻完成的

> 值得注意的是帮助程序`log()`方法会添加自应用程序加载到每条消息以来的微秒数,这在之后的代码示例中会给更多的提示

> 在observer中有next和complete两个方法分别用来`emit`新的值和将Observable完成,虽然看起来`subscribe`和`then`看起来像是孪生兄弟,但是从本质上来说他们相隔千里

> 说一句题外话，Observable分为有无限和有限的Observables，正如名称所暗示的那样，有限可观察量会返回`一定数量`的结果，而无限可观测量可以`永远持续下去`