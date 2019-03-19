---
title: "Redux-Middleware"
description: "探究redux中Middleware的实现及调用，考量一下其中的函数式编程思想。"
image: relay.jpg
keywords: "redux middleware functional programming"
data: 2018-08-01
draft: false
---
redux作为前端广为人知的状态管理库，有着诸多优秀的思想及代码实现值得学习，相关解析网上也是一应俱全。  
本篇从redux Middleware的实现、调用这个角度切入，学习一下redux中优秀的函数式编程实现。

相关基础知识：  

 - ES6箭头函数，剩余运算符，展开运算符等
 - Functional programming常规概念
 - applymiddleware API使用  
 - redux源码  
 - redux 异步派发方案，如redux-thunk,redux-promise等
 - redux Middleware源码，本篇以[redux-thunk](https://github.com/reduxjs/redux-thunk)举例

来看一看redux源码：
{{< highlight javascript >}}
  function createStore(reducer, preloadedstate, enhancer){
      if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
        enhancer = preloadedState
        preloadedState = undefined
      }

    if (typeof enhancer !== 'undefined') {
      if (typeof enhancer !== 'function') {
        throw new Error('Expected the enhancer to be a function.')
      }
      //局部调用
      return enhancer(createStore)(reducer, preloadedState)
    }
    ...
  }
{{< /highlight >}}
可以看到初次createStore时，倘若传入了"enhancer"参数——例如applyMiddleware，redux就会中止当前创建行为。  
并且将createStore函数本身、reducer、preloadedState通过两次局部调用，传入"enhancer"。  

再来看看applyMiddleware做了什么：
{{< highlight javascript >}}
function applyMiddleware(...middlewares){
  return createStore => (...args) => {
    ...
  }
}

const store=createStore(reducer, applyMiddle(middleware1, middleware2));
{{< /highlight >}}
很巧妙的柯里化(curry)函数实现，结合applyMiddleware API的使用，可以发现原来在常规使用redux中createStore时，我们就进行了applyMiddleware的第一次局部调用。

    什么是局部调用？  
    简单表述就是将一个需要多个参数的函数，变式成一个每次接收一个参数的柯里化函数。

{{< highlight javascript >}}
//这是一个普通的函数。
function test(a, b, c, d){
  return a + b + c + d;
}

//这是一个柯里化函数。
function test(a){
  return function (b){
    return function (c){
      return function (d){
        return a + b + c + d;
      }
    }
  }
}

const add_B = test(a);
const add_C = add_B(b);
const add_D = add_C(c);

const result1 = add_D(d);
const result2 = test(a)(b)(c)(d);

result1 === result2; //true

const test = a => b => c => d => a + b + c + d;
{{< /highlight >}}

看来柯里化函数确实有些妙用，通过每一次的局部调用，来实现函数的“预加载”。通过分批次传入参数，每次都可以得到一个“记住”以前参数的新函数。  
这样可以更加灵活应用函数的同时，还提升了函数的可移植性。 

“提升可移植性”在redux中最直观的应用就是由createStore函数中获取createStore本身，而后借助返回新函数来接收开发者传入的reducer、preloadedState。  

使用ES2015箭头函数的写法，能让代码更加的优雅。  

回到applyMiddleware的思路上来，为什么在初次局部调用接收到所有Middleware后，要在createStore函数中再接收一遍createStore函数呢？

{{< highlight javascript >}}
function applyMiddleware(...middlewares){
  return createStore => (...args) => {
    ...
  }
}

const store=createStore(reducer, applyMiddleware(middleware1, middleware2));
{{< /highlight >}}

{{< highlight javascript >}}
function createStore(reducer, preloadedstate, enhancer){
  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }
    return enhancer(createStore)(reducer, preloadedState)
  }
  ...
}
{{< /highlight >}}  

这里需要明确一个主要概念：reudx中的Middleware本质是对"dispatch"这一行为的改造。  

阮一峰[Redux 入门教程（二）：中间件与异步操作](http://www.ruanyifeng.com/blog/2016/09/redux_tutorial_part_two_async_operations.html)一文中有一段易于理解的阐述：

> 为了理解中间件，让我们站在框架作者的角度思考问题：如果要添加功能，你会在哪个环节添加？  
> 1. Reducer：纯函数，只承担计算 State 的功能，不合适承担其他功能，也承担不了，因为理论上，纯函数不能进行读写操作。  
> 2. View：与 State 一一对应，可以看作 State 的视觉层，也不合适承担其他功能。  
> 3. Action：存放数据的对象，即消息的载体，只能被别人操作，自己不能进行任何操作。  

因此dispatch的“传递”本质不能被改变。  
Store该创建还是要创建，Middleware只是对原本Store.dispatch做了封装。  
这样就有了一个思路：Middleware是高阶函数，接收的该是"dispatch"这一行为，返回的也该是"dispatch"这一行为。  
如此一来，多个Middleware可以被串联使用，思路上也不会影响最终的"dispatch"行为。  

![redux-Middleware](/img/notes/middleware.png)  

而封面所展示的图片，也算是Middleware比较形象的一个比喻。每一个Middleware就像一个运动员，接力棒(action)经由每个Middleware处理后再转交给下一个。  
{{< highlight javascript >}}
//redux中对applyMiddleware的两次局部调用。
return enhancer(createStore)(reducer, preloadedState)

function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
    //args === [reducer,preloadedState]
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        `Dispatching while constructing your middleware is not allowed. ` +
          `Other middleware would not be applied to this dispatch.`
      )
    }
    //伪·store.
    const middlewareAPI = {
      //真·store.getState.
      getState: store.getState,
      //伪·dispatch，非store.dispatch.
      dispatch: (...args) => dispatch(...args)
    }
    //每个Middleware的第一次局部调用，接收“伪·store”。
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
}
{{< /highlight >}}
果然applyMiddleware创建了新的store，并通过redux中对applyMiddleware的两次局部调用，又回到了createStore这一操作上。  
新的createStore没有enhancer参数，一切照常。  
createStore，准备封装store.dispatch，代码解耦一气呵成。看来代码解耦就是柯里化函数一个显而易见的好处。  

    近期更新的redux对applyMiddleware中所封装的dispatch做了更严谨的判断。  
    为了防止编写Middleware的开发者在Middleware中粗暴调用原store.dispatch从而掉入死循环，新dispatch初始值是一个会抛错的函数。  
    通篇解读完成后“为什么会掉入死循环”这个问题也就迎刃而解，这里埋个伏笔。  

继续，applyMiddleware创建了一个命名为"middlewareAPI"的“伪·store”，通过map传递给所有Middleware这个“伪·store”。  
思路顺到Middleware上。所有符合redux规范的Middleware大体都是这样一个结构：

{{< highlight javascirpt >}}
store => next => action => next(action);
{{< /highlight >}}

以[redux-thunk](https://github.com/reduxjs/redux-thunk/blob/master/src/index.js)举例，每个Middleware都会接收“真·store.getState”和“伪·dispatch”，便于获取数据(getState)和异步派发(dispatch)使用。  
Middleware完成第一次局部调用后，返回了一个这样结构的函数：

{{< highlight javascript >}}
next => action => next(action);
{{< /highlight >}}

所以"chain"中是每一个完成初次局部调用的Middleware，随后被解构传入compose：  

{{< highlight javascript >}}
const chain = middlewares.map(middleware => middleware(middlewareAPI))
dispatch = compose(...chain)(store.dispatch)
{{< /highlight >}}

好，接下来**重点**探究一下一直为人称道的compose.  

{{< highlight javascript >}}
function compose(...funcs) {
  //无Middleware传入。
  if (funcs.length === 0) {
    return arg => arg
  }
  //一个Middleware.
  if (funcs.length === 1) {
    return funcs[0]
  }
  //多个Middleware.
  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
{{< /highlight >}}

可以看到compose做了健全的判断：  
 1. 无Middleware传入情况下，将第二次局部调用所接收的store.dispatch原样不动赋值给所谓的“伪dispatch”。  
 2. 单一Middleware情况下，原样不动将Middleware返回。  
 3. 多个Middleware情况下，利用Array.prototype.reduce对多个Middleware进行了酷炫的拼接操作。  

{{< highlight javascript >}}
//compose拼接的函数——即拼接Middleware
funcs.reduce((a, b) => (...args) => a(b(...args)))

//执行过一次——即局部调用过一次的Middleware
next => action => next(action);

function applyMiddleware(...middlewares){
  dispatch = compose(...chain)(store.dispatch)
  return {
    ...store,
    dispatch
  }
}
{{< /highlight >}}
那么结合compose的三种情况，代入初次局部调用完成的Middleware，在applyMiddleware中将全部逻辑串联起来：  

#### 无Middleware
“伪·dispath”被赋值store变成“真·dispatch”，store被原本返回，一切如常。

#### 单一Middleware
{{< highlight javascript >}}
//compose(...chain) === next => action => next(action);
next => action => {
  //next === store.dispatch;
  next(action);
}
//这就是改造完成的dispatch.
action => dispatch(action);
{{< /highlight >}}
原本的Middleware接收store.dispatch为"next"，即第二个参数。  
返回一个接收"action"的函数，在这个函数中有"store.dispatch(action)"这一行为。  
这个“接收'action'的函数”，就是改造完成的"dispatch"，"action"就是开发者派发的原action.

#### 多个Middleware
{{< highlight javascript >}}
//Middleware会被拼接成这种形态
(...args) => Middleware_1(Middleware_2(Middleware_3(...args)));

compose(...chain) === (...args) => Middleware_1(Middleware_2(Middleware_3(...args)));

//伪·dispath，就是改造完成的dispatch.
dispatch = compose(...chain)(store.dispath);

//Middleware拼接完成后接收store.dispatch，就是这种形态。
Middleware_1(Middleware_2(Middleware_3(store.dispatch)));

//结合每一个执行过一次的Middleware。
next => action => next(action);
Middleware_1(Middleware_2(Middleware_3(store.dispatch)));
{{< /highlight >}}

这一小节最后两行代码是一切的聚合点，比较绕。能够将这两行完全串联起来，前面所有的逻辑就全部联动起来了。  

可以看到，首先是Middleware_3真正接收到store.dispatch，next(action)这个行为就是store.dispatch(action).  
Middleware_2接收到Middleware_1(store.dispatch)的返回值，变成Middleware_2(action => store.dispatch(action)).  
Middleware_2中的"next"就是action => store.dispatch(action)这样一个函数。  
以此类推，每个Middleware真正的生效顺序是Middleware_3、Middleware_2、Middleware_1。  
{{< highlight javascript >}}
const store = createStore(reducer, applyMiddleware(Middleware_1, Middleware_2, Middleware_3));
{{< /highlight >}}

---

### 归纳
前面埋下了一个“为什么在Middleware中直接调用原store.dispatch会掉入死循环？”的伏笔。  

![洋葱圈模型](/img/notes/洋葱圈儿.jpg)

redux是flux架构的实现，原则上数据流动是单向的。  
applyMiddleware的最终结果就是覆盖原store.dispatch，不过在applyMiddleware执行诸多Middleware时还是会用到原store.dispatch——毕竟最终还是要将action好好儿的还给store.  
一旦在Middleware中直接调用原store.dispatch，无异于又从外面“来”了一遍。  
结果又到了你调用原store.dispatch的Middleware的操作点，就跳进死循环了。  

而且redux的Middleware规范中提到：遵循next => action => next(action)这样的结构，且一定要有"next(action)"这样一个操作。  
经由前面代码分析可知，如果有一个Middleware“捣乱”，data在Middleware中的流动就会“断裂”。  
代码场景大体就会是这样：
{{< highlight javascript >}}
action => {
  ...
  /**
  * store:“???”
  * 没有执行‘next(action)’即没有进行真正的dispatch.
  * 但并不一定会引发语法错误，这是显而易见的。
  */
}
{{< /highlight >}}
这也是很多Middleware会在文档中央求开发者尽可能将自己“放在”最右边，很重要的一个原因是怕上一个Middleware“不讲究”。

还有一个遗漏的点，即redux的异步方案。  
{{< highlight javascript >}}
let dispatch = () => {
  throw new Error(
    `Dispatching while constructing your middleware is not allowed. ` +
    `Other middleware would not be applied to this dispatch.`
  )
}
const middlewareAPI = {
  getState: store.getState,
  dispatch: (...args) => dispatch(...args)
}
const chain = middlewares.map(middleware => middleware(middlewareAPI))
//改造完成的dispatch
dispatch = compose(...chain)(store.dispatch)
{{< /highlight >}}
applyMiddleware中对Middleware的首次局部调用传了个“伪·dispatch”。  
“伪·store”中利用闭包保存了“伪·dispatch”，对就是那个初始值是会抛错的函数。  
当Middleware需要异步派发action时，直接调用原store.dispatch不会出错，但是会忽略掉所有的Middleware.  
所以需要Middleware在进行异步派发时使用被改造完成的dispatch，所以"middlewareAPI"使用闭包保存“伪·dispatch”。  
届时异步操作执行时会调用到改造完成的dispath.  

惊艳~

---

### 总结
回过头来再回顾一下，redux applyMiddleware中有着诸多函数式编程思想的明显应用。  

 - curry
 - chain
 - compose
 - pure function(Array.prototype.map, Array.prototype.reduce...)

柯里化函数的灵活局部调用、chain链的灵活拼接、使用Array.prototype.reduce实现的酷炫compose等等...  
且代码本身不需过多的注释——函数本身就是良好的注释。  
学习函数式编程相关时，需要解冻一种思维：  

    函数可以作为参数传给另一个函数，当然也可以返回新的函数，多个函数可以拼接成一个更强大的函数，只需一次调用传递真正值就可以完成诸多操作。

而纯函数(pure function)意指不会对原值产生破坏的函数，举个形象的例子就是：

 - Array.prototype.map
 - Array.prototype.splice

map是纯函数，splice总是对原值进行直接操作，是危险函数。  
对于纯函数的一点重要要求，就是相同的输入总能得到相同的输出。  

compose操作，很常见的一种形态就是“函数饲养”——将两个或多个函数组合成新的函数。  
applyMiddleware中就是很纯熟的应用。  

---

### In the end
笔者近期初学函数式编程相关，的确在很多方面收获了新的东西。  
redux作为业界公认的“艺术品”，真的是每次解读都有不同的收获。作为函数式编程的学习案例，也能有其贴切之处。  

文字中使用了一些不严谨、比较“接地气”的表述，如有错误万望指出。  
对于其它相关知识或错误，欢迎指点。
