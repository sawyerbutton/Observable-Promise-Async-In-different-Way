# Observable-Promise-Async-In-different-Way

## Observable, Promise 和 Async await 三种不同的异步方式之我见 基于Angular

### 前言

> Angular作为一个前端框架之所以有劝退效果的原因之一是 Angular坚持使用 Observer/Observable 设计模式, 随时随地异步返回的都是Rxjs的Observable对象而不是开发者所熟悉的Promise

> 作为一个开发者当他试图开始拿起Angular时面对一系列的新概念是非常令人沮丧的,开发者的直觉反应是使用Observables提供的非常方便的toPromise()方法将Observable转换成Promise进行使用, 但是这样是本末倒置的行为, 抛弃了Observable所既存的巨大优势

> 希望通过此文帮助自己巩固Angular的基本

#### 为什么使用Observable而抛弃Promise

> 简而言之有几条理由, 我们之后溯源而来
1. Observables允许取消正在进行的任务(如果不再需要对`服务器的HTTP请求`或`需求其他一些昂贵的异步操作的结果`,则`Observable的订阅允许取消订阅`,而`Promise即使没有通知的需求最终也会调用成功或失败的回调`)
2. Observables允许返回多个物件(之所以使用物件是因为返回的`东西`是不确定类型的)
3. 单个Observables实例允许存在多个订阅者
4. Observables简化了重试的过程
5. Observable类似于Stream允许传递零个或多个事件并为每个事件调用回调
6. Observable可以从事件等其他来源创建
7. 当异步操作完成或失败时,Promise只会处理单个事件
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

> 假设希望构建一个搜索功能,该功能可以在键入时立即显示结果,就像各大搜索引擎一样, 这样的需求一定不陌生,但是这实际上充满了挑战

1. 搜索不希望用户每次按下某个键时都会触发服务器端点搜索,那样会导致服务端充斥着一系列的http请求,事实上对于后端服务器而言希望在用户停止键入时发送搜索请求而不是每次按键时都发送请求

2. 对于持续的搜索不希望使用相同的查询参数重复对后端服务进行请求

3. 能够处理无序的响应,比如当同一时间存在多个请求时必须考虑请求的响应以意想不到的顺序返回的情况(想象一下我们分先后快速发出两个搜索请求,一个查询汽车时间一个查询火车时间,这样在网络中存在两个请求,但是因为搜索服务分片等后端原因或网络延迟原因,查询火车的请求先于查询汽车的请求回来,这就很尴尬了)

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

> 构造器中注入Jsonp服务,以针对具有给定搜索内容的search API发出GET请求

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

> 在组件中注入了searchService并通过`search`方法将其功能暴露给模板,模板将`keyUp`事件绑定其上

> 将searchService的搜索方法返回的promise结果展开,并将其作为一个简单的字符串数组暴露给模板,以便使用`*ngFor`循环建立列表

> 当使用Promise时并非无法实现之前的某些边际条件只是为了照顾到边际条件会在view层写很多的页面判断逻辑,但是仍旧无法保证返回数据的先后问题

##### 而当你使用Observable时一切都会变得简单起来

> 首先按照期望修改代码实现`不在每次键入的时候`发送搜索请求,而是在`用户停止键入400ms时`才发送搜索请求

> 为此需要取得一个带有用户输入的搜索词的`Observable<string>`,通过利用Angular的formControl指令代替手动绑定`keyUp`事件

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

> 在代码的背后,`term`属性会自动暴露名为`valueChanges`实际是`Observable<string>`的属性

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

> 这个spa直接运行的结果很直白,会直接展示出`Hello, observable world!`,因为Observable是立刻完成的

> 值得注意的是帮助程序`log()`方法会添加自应用程序加载到每条消息以来的微秒数,这在之后的代码示例中会给更多的提示

> 在observer中有next和complete两个方法分别用来`emit`新的值和将Observable完成,虽然看起来`subscribe`和`then`看起来像是孪生兄弟,但是从本质上来说他们相隔千里

> 说一句题外话,Observable分为有无限和有限的Observables,正如名称所暗示的那样,有限可观察量会返回`一定数量`的结果,而无限可观测量可以`永远持续下去`

> `事实上根本不需要像上述代码中自己创建Observable并发出值,上面的代码可以用更简洁的方式代替`

```typescript
this.doAsyncObservableThing = of('Hello, observable world!');
```
 
> 需要注意的是在使用of操作符时需要提前导入, 例如

```typescript
import {of} from 'rxjs';
```

> 值得庆幸的事情是开发者不需要取消订阅有限的可观察量,RxJS会帮助你处理，比如

```typescript
this.doAsyncObservableThing = new Observable(observer => {
  observer.next('Started');
  setTimeout(() => {
    observer.next('Hello, observable world!');
  }, 1000);
  setTimeout(() => {
    observer.next('Done');
    observer.complete();
  }, 2000);
});
```

> 程序输出的结果是

```bash
0: Started
1001: Hello, observable world!
2001: Done
```

> 在上面的例子中我们使用了`setTimeout()`来进行延迟触发，在实际的应用场景中，更好地解决方案是通过`delay()`方法来实现

> 因为使用subscribe方法,Observable将保持订阅并且每次都按照预期调用日志函数，但是却无法知晓observable何时完全完成，即使这是一个finite的Observable因为最后调用了`observer.complete();`方法

> 为了显式的知晓Observable完成,可以换一种方式

```typescript
this.doAsyncObservableThing = new Observable(observer => {
  observer.next('Started');
  setTimeout(() => {
    observer.next('Hello, observable world!');
  }, 1000);
  setTimeout(() => {
    observer.complete();
  }, 2000);
});
this.doAsyncObservableThing.forEach(
  this.log
).then(() => {
  this.log('Done');
});
```

> `forEach()`方法将把Observable转换成Promise，亦即可以使用then标识出Observable完成的时间，当然这只是为了说明而并不是一个开发的方案

#### 链式Observable

> 当开始使用Observable时,最容易困惑的是如何将Observables串联在一起

> 一个很常见的场景是顺序调用多个http请求，并且在请求响应之间需要做一些处理

> 对此的最简单也是最天真的解决方案是简单地在subscribe内部subscribe，就像是回调地狱一样，比如

```typescript
this.doAsyncObservableThing = new Observable(observer => {
  setTimeout(() => {
    observer.next('Hello, observable world!');
    observer.complete();
  }, 1000);
});
this.doAsyncObservableThing.subscribe((val) => {
  this.log(val);
  // 在subscribe的内部继续subscribe
  this.doAsyncObservableThing.subscribe((val) => {
    this.log(val);
  });
});
```

> 作为结果

```bash
1001: Hello, observable world!
2001: Hello, observable world!  
```

> 就像promise出现之前的面条回调一样，代码会随着需求嵌套的增多变得越来越难以维护

> 当然可以选择使用`toPromise`的方式将Observable转换成Promise进行优化，但这并不`Observable`

> 事实上，可选的方案根据需求大致存在几种

1. 立即启动所有subscribe任务，逐个获取结果并合并(利用`merge`操作符实现)

```typescript
import {merge} from 'rxjs/operator'
this.doAsyncObservableThing('First')
.merge(this.doAsyncObservableThing('Second'))
.subscribe((v) => {
  this.log(v);
});
```

```bash
// 结果为
1002: First
1002: Second
```

> 关键点是`subscribe()`作为回调被调用两次，两次调用同一时刻开始同一时刻结束，first在前，second在后

2. 当Observable是序列实现的时候，使用`async await`

> 通过`toPromise`将Observable转换成Promise后，使用`async await`关键字将异步代码转化为同步的形式撰写

> 使用该方法需要将方法从`constructor()`中转移到`ngOnInit()`,因为构造函数内部不支持异步函数

> async是一个关键字，表示允许方法使用`await`关键字，并且返回一个`promise`

> await关键字暂停async函数的执行，直到`promise被解决或拒绝为止`,await必须始终跟随一个`返回值为promise的表达式`,具体来看

```typescript
this.log(await this.doAsyncObservableThing('First').toPromise());
this.log(await this.doAsyncObservableThing('Second').toPromise());
```

> 使用async await后，对`this.log()`的调用将被暂停，直到计算出涉及await的表达式的promise之后才会继续执行,导致的结果是

```bash
1005: First
2009: Second
```

> 正如我们所期待的那样，第二个Observable将只能在第一个Observable转换成的Promise出结果后才会被执行

3. 使用操作符`flatMap`和`forkJoin`

> 需要明确的是在Angular中，HttpClient的方法返回的是`cold Observable`

> 所谓`cold Observable`指Observable在subscribe时才会开始运行的Observable

> 而所有对于Observable流的操作都必须在`subscribe`方法被调用前之前进行

> 为了循序渐进将会从头开始举例子，下述是一个http service和调用该service的component

```typescript
@Injectable()
export class AuthorService {

  constructor(private http: HttpClient){}

  get(id: number): Observable<any> {
    return this.http.get('/api/authors/' + id)
  }
}
```

```typescript
@Component({
  selector: 'app-author',
  templateUrl: './author.component.html'
})
export class AuthorComponent implements OnInit {

  constructor(private authorService: AuthorService) {}

  ngOnInit() {
    this.authorService.get(1).subscribe((data: any) => {
      console.log(data);
    });
  }
}
/* return values:
{
  id: 1,
  first_name: 'sawyer',
  last_name: 'button'
}
*/
```

> 上述是最基本的http service调用，接下来来一些高级的

> 比如:并行组合Observable

> 假设想要获取作者及其书籍的数据，但为了获取书籍的数量需要调用不同的端点，例如`/authors/1/books`

> 现在希望调用两个端口并将响应组合在一起, 这就需要`forkJoin`操作符(其功能类似于Promise.all());

```typescript
getAuthorWithBooks(id: number): Observable<any> {
  return forkJoin([
    this.http.get('/api/authors/' + id),
    this.http.get('/api/authors/' + id + '/books')
  ]).map((data: any[]) => {
    let author: any = data[0];
    let books: any[] = data[1];
    author.books = books;
    return author;
  });
}
/* return values:
{
  id: 1,
  first_name: 'Daniele',
  last_name: 'Ghidoli'
  books: [{
    id: 10,
    title: 'Awesome book',
    author_id: 1
  }]
}
*/
```

> `forkJoin`返回一个`Array`，其中包含`已连接的Observables`的结果,之后可以在map函数中对其进行操作以满足需要

> 再比如: `将Observables串联组合`

> 在需要从书中获取作者信息的场景下，应该首先得到书籍的数据，在拿到书籍数据中的authorId后再调用author的接口

> 此时需要使用`flatMap`操作符，它类似于通常的`map`操作符，不同之处在于可以链接两个Observable并返回一个新的Observable

```typescript
getBookAuthor(id: number): Observable<any> {
  return this.http.get('/api/books/' + id).pipe(
        flatMap((book: any) => {
            return this.http.get('/api/authors/' + book.author_id)
    });
  )
}

/* return values:
{
  id: 1,
  first_name: 'Daniele',
  last_name: 'Ghidoli'
}
*/
```

> 假设需要一并获得书籍和作者的信息时:

```typescript
getBookAuthor(id: number): Observable<any> {
  return this.http.get('/api/books/' + id).pipe(
        flatMap((book: any) => {
            return this.http.get('/api/authors/' + book.author_id)
            .map((author: any) => {
                 book.author = author;
                return book;
            })
    });
  )
}
/* return values:
{
  id: 10,
  title: 'Awesome book',
  author_id: 1
  author: {
    id: 1,
    first_name: 'Daniele',
    last_name: 'Ghidoli'
  }
}
*/
```

> 当情境变得更复杂一些,需要获得许多本书的书籍信息和作者信息时, 需要融合`forkJoin`与`flatMap` 操作符

```typescript
getBooksWithAuthor(): Observable<any[]> {
  return this.http.get('/api/books/')
    .pipe(
    flatMap((books: any[]) => {
      if (books.length > 0) {
        return forkJoin(
          books.map((book: any) => {
            return this.http.get('/api/authors/' + book.author_id)
              .map((author: any) => {
                book.author = author;
                return book;
              });
          });
        );
      } else {
        return Observable.of([]);
      }
    });
    )
}

/* return values:
[{
  id: 10,
  title: 'Awesome book',
  author_id: 1
  author: {
    id: 1,
    first_name: 'Daniele',
    last_name: 'Ghidoli'
  }
},
{
  id: 11,
  title: 'Another awesome book',
  author_id: 2
  author: {
    id: 2,
    first_name: 'Jeff',
    last_name: 'Arese'
  }
}]
*/
```

> 虽然上述代码看起来很复杂，但原理并不难懂:在获取书籍列表之后使用flatMap将之前的调用与forkJoin的结果合并

> 只当书籍不为空时调用，否则我返回一个包含空数组的Observable

> 再举一个例子: 通过一本书获取该书的作者和编辑信息

```typescript
getBookWithDetails(id: number): Observable<any> {
  return this.http.get('/api/books/' + id)
    .pipe(
        flatMap((book: any) => {
            return forkJoin(
            Observable.of(book),
            this.http.get('/api/authors/' + book.author_id),
            this.http.get('/api/editors/' + book.editor_id)
      ).map((data: any[]) => {
          let book = data[0];
          let author = data[1];
          let editor = data[2];
          book.author = author;
          book.editor = editor;
          return book;
        });
    });
    )
}
/* return values:
{
  id: 10,
  title: 'Awesome book',
  author_id: 1,
  editor_id: 42
  author: {
    id: 1,
    first_name: 'Daniele',
    last_name: 'Ghidoli'
  }, 
  editor: {
    id: 42,
    name: 'Universe Editor'
  }
}
*/
```

> 值得注意的是，我们利用`of`操作符将book对象转换为Observable以实现forkjoin


#### 关于`route.queryParams`的坑

> 当开发者希望读取当前路由中的查询参数时，必须使用route.queryParams: Observable

> 但是对该Observable使用`toPromise()`方案进行，Observable将会永远卡在这里

> queryParams是一个infinite Observable,订阅它会导致只有在每次查询参数更改时才会调用subscribe，而不是在subscribe时获取当前查询参数

> 这意味着`observer.complete()`将永远不会被其中的内部机制调用，这意味着它对应的Promise将永远不会resolve或reject，await操作符也就失去了作用

> 更多内容涉及到Hot Observable和 Cold Observable

> 如果是应对只希望获取查询参数一次的状况，可以通过导入`take`或`first`运算符变相实现该功能，例如:

```typescript
import {take} from 'rxjs/opeartor';
const queryParams = await this.route.queryParamMap.take(1).toPromise();
// access queryParams as needed
```