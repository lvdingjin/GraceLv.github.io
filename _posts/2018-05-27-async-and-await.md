---
layout: post
title: "promise、async和await之间那点事"
description: ""
category: tech
tags: []
modify: 2018-05-27 17:25:29
---

update: 2018-05-27


故事要从一道今日头条的笔试题说起～ </br>
题目来源：[https://juejin.im/post/5b03e79951882542891913e8]()

```
async function async1(){
	console.log('async1 start')
	await async2()
	console.log('async1 end')
}
async function async2(){
	console.log('async2')
}
console.log('script start')
setTimeout(function(){
	console.log('setTimeout') 
},0)  
async1();
new Promise(function(resolve){
	console.log('promise1')
	resolve();
}).then(function(){
	console.log('promise2')
})
console.log('script end')
```
求打印结果是什么？

相信是个前端都知道啦，这道题目考的就是js里面的事件循环和回调队列咯～</br>今天题主假设看客都已经了解了setTimeout是宏任务会在最后执行的前提（因为它不是今天要讨论的重点），我们主要来讲讲**promise**、**async**和**await**之间的关系。

先上正确答案：

```
script start
async1 start
async2
promise1
script end
promise2
async1 end
setTimeout
```
事实上，没有在控制台执行打印之前，我觉得它应该是这样输出的：

```
script start
async1 start
async2
async1 end
promise1
script end
promise2
setTimeout
```
为什么这样认为呢？因为我们（粗浅地）知道await之后的语句会等await表达式中的函数执行完得到结果后，才会继续执行（可以看作是一个同步的等待）。

MDN是这样描述await的：
> async 函数中可能会有 await 表达式，这会使 async 函数暂停执行，等待表达式中的 Promise 解析完成后继续执行 async 函数并返回解决结果。

这话是没错，但是当场景变得复杂（很多promise啊async啊结合一起使用）时，光知道这个概念就有点不够用了，需要了解更多的细节和深入理解其中的概念。

### async ###
> async function 声明将定义一个返回 AsyncFunction 对象的异步函数。
> 
> 当调用一个 async 函数时，会返回一个 Promise 对象。当这个 async 函数返回一个值时，Promise 的 resolve 方法会负责传递这个值；当 async 函数抛出异常时，Promise 的 reject 方法也会传递这个异常值。

所以你现在知道咯，使用 **async** 定义的函数，当它被调用时，它返回的其实是一个Promise对象。</br>
我们再来看看 **await** 表达式执行会返回什么值。

### await ###
> 语法：[return_value] = await expression;
> 
> 表达式（express）：一个 Promise 对象或者任何要等待的值。
> 
> 返回值（return_value）：返回 Promise 对象的**处理结果**。如果等待的不是 Promise 对象，则返回该值本身。

所以，当await操作符后面的表达式是一个Promise的时候，它的返回值，实际上就是Promise的回调函数resolve的参数。

明白了这两个事情后，我还要再啰嗦两句。我们都知道Promise是一个立即执行函数，但是他的成功（或失败：reject）的回调函数resolve却是一个异步执行的回调。当执行到resolve()时，这个任务会被放入到回调队列中，等待调用栈有空闲时事件循环再来取走它。

#### 终于进入正文：解题 ####

好了铺垫完这些概念，我们回过头看上面那道题目（建议一边对着题目一边看解析我怕我讲的太快你跟不上啊哈哈😂）。

执行到 async1 这个函数时，首先会打印出“async1 start”（这个不用多说了吧，async 表达式定义的函数也是立即执行的）；

然后执行到 await async2()，发现 async2 也是个 async 定义的函数，所以直接执行了“console.log('async2')”，同时async2返回了一个Promise，**划重点：此时返回的Promise会被放入到回调队列中等待，await会让出线程（js是单线程还用我介绍吗），接下来就会跳出 async1函数 继续往下执行。**

然后执行到 new Promise，前面说过了promise是立即执行的，所以先打印出来“promise1”，然后执行到 resolve 的时候，resolve这个任务就被放到回调队列中（前面都讲过了上课要好好听啊喂）等待，然后跳出Promise继续往下执行，输出“script end”。

接下来是重头戏。同步的事件都循环执行完了，调用栈现在已经空出来了，那么事件循环就会去回调队列里面取任务继续放到调用栈里面了。

这时候取到的第一个任务，就是前面 async1 放进去的Promise，执行Promise时发现又遇到了他的真命天子resolve函数，**划重点：这个resolve又会被放入任务队列继续等待，然后再次跳出 async1函数 继续下一个任务。**

接下来取到的下一个任务，就是前面 new Promise 放进去的 **resolve回调** 啦 yohoo～这个resolve被放到调用栈执行，并输出“promise2”，然后继续取下一个任务。

后面的事情相信你已经猜到了，没错调用栈再次空出来了，事件循环就取到了下一个任务：**历经千辛万苦终于轮到的那个Promise的resolve回调！！！**执行它（啥也不会打印的，因为 async2 并没有return东西，所以这个resolve的参数是undefined），此时 await 定义的这个 Promise 已经执行完并且返回了结果，所以可以继续往下执行 async1函数 后面的任务了，那就是“console.log('async1 end')”。

### 总结 ###
总结下来这道题目考的，其实是以下几个点：

1. 调用栈
1. 事件循环
1. 任务队列
1. promise的回调函数
1. async表达式的返回值
1. await表达式的作用和返回值

理解了这些，自然就明白了为什么答案是这样（答出笔试题还要分析给面试官原因哈哈哈）～

关于**调用栈、事件循环、任务队列**可以[点这里](https://github.com/xitu/gold-miner/blob/master/TODO/how-javascript-works-event-loop-and-the-rise-of-async-programming-5-ways-to-better-coding-with.md)了解更详细的描述。</br>
为了方便大家直接贴图:
![tupian](https://camo.githubusercontent.com/bd3cc88e02a70dbd46694ef8ad2f0b5741725d7e/68747470733a2f2f63646e2d696d616765732d312e6d656469756d2e636f6d2f6d61782f3830302f312a4641394e47784e42362d76316f4932714745746c52512e706e67)

关于async和await的执行顺序[这里](https://segmentfault.com/a/1190000011296839)也有很详细的分析可以参考～

资料参考：</br>
[https://segmentfault.com/a/1190000011296839]()</br>
[https://github.com/xitu/gold-miner/blob/master/TODO/how-javascript-works-event-loop-and-the-rise-of-async-programming-5-ways-to-better-coding-with.md]()

