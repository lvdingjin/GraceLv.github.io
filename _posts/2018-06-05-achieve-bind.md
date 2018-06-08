---
layout: post
title: "手动实现bind函数（附MDN提供的Polyfill方案解析）"
description: ""
category: tech
tags: []
modify: 2018-06-05 23:18:29
---

update: 2018-06-08

为什么要自己去实现一个bind函数？
> bind()函数在 ECMA-262 第五版才被加入；它可能无法在所有浏览器上运行。

所以，为了理想主义和世界和平（所有浏览器上都能随心所欲调用它），必要的时候需要我们自己去实现一个`bind`。那么，一个`bind`函数需要具备什么功能呢？

### bind函数的核心作用：绑定this、初始化参数
绑定this、定义初始化参数是它存在的主要意义和价值。MDN对它的定义如下：
> 语法：fun.bind(thisArg[, arg1[, arg2[, ...]]])
> 
> bind()方法创建一个新的函数, 当被调用时，将其this关键字设置为提供的值（thisArg）。
> 
> 被调用时，arg1、arg2等参数将置于实参之前传递给被绑定的方法。
> 
> 它返回由指定的this值和初始化参数改造的原函数拷贝。

鉴于这两个核心作用，我们可以来实现一个简单版看看：

```
if (!Function.prototype.bind) {
  Function.prototype.bind = function (oThis) {
  	if (typeof this !== 'function') {
      return
    }
  
    let self = this
    let args = Array.prototype.slice.call(arguments, 1)
    return function () {
      return self.apply(oThis, args.concat(Array.prototype.slice.call(arguments))) //这里的arguments是执行绑定函数时的实参
    }
  }
}
```
由于`arguments`是类数组对象，不拥有数组的`slice`方法，所以需要通过`call`来将`slice`的`this`指向`arguments`。`args`就是调用`bind`时传入的初始化参数（剔除了第一个参数`oThis`）。将`args`与绑定函数执行时的实参`arguments`通过`concat`连起来作为参数传入，就实现了`bind`函数初始化参数的效果。

`bind`函数的另外一个也是最主要的作用：绑定`this`指向，就是通过将调用`bind`时的`this`（`self`）指向指定的`oThis`来完成。这样当我们要使用`bind`绑定某个对象时，执行绑定函数，它的`this`就永远固定为指定的对象了～

### 遇到new操作符的时候呢

到这里，我们已经可以用上面的版本来使用大部分场景了。但是～

但是，这种方案就像前面说的，它会永远地为绑定函数固定`this`为指定的对象。如果你仔细看过MDN关于`bind`的描述，你会发现还有一个情况除外：
> thisArg：当使用new 操作符调用绑定函数时，该参数无效。
> 
> 一个绑定函数也能使用new操作符创建对象：这种行为就像把原函数当成构造器。提供的 this 值被忽略，同时调用时的参数被提供给模拟函数。

我们可以通过一个示例来试试看原生的`bind`对于使用`new`的情况是如何的：

```
function animal(name) {
    this.name = name
}
let obj = {}

let cat = animal.bind(obj)
cat('lily')
console.log(obj.name)  //lily

let tom = new cat('tom')
console.log(obj.name)  //lily
console.log(tom.name)  //tom
```
试验结果发现，`obj.name`依然是`lily`而没有变成`tom`，所以就像MDN描述的那样，如果绑定函数`cat`是通过`new`操作符来创建实例对象的话，`this`会指向创建的新对象`tom`，而不再固定绑定指定的对象`obj`。

而上面的简易版却没有这样的能力，它能做到的只是永久地绑定指定的`this`（有兴趣的胖友可以在控制台使用简易版`bind`试下这个例子看看结果）。这显然不能很好地替代原生的`bind`函数～

那么，如何才能区分绑定函数有没有通过`new`操作符来创建一个实例对象，从而进行分类处理呢？

### 区分绑定函数是否使用new，分类处理

我们知道检测一个对象是否通过某个构造函数使用`new`实例化出来的最快的方式是通过 **instanceof**：

`A instanceof B //验证A是否为B的实例`

那么，我们就可以这样来实现这个`bind`：

```
if (!Function.prototype.bind) {
  Function.prototype.bind = function (oThis) {
  	if (typeof this !== 'function') {
      return
    }
    
    let self = this
    let args = Array.prototype.slice.call(arguments, 1)
    let fBound = function() {
    	let _this = this instanceof self ? this : oThis //检测是否使用new创建
        return self.apply(_this, args.concat(Array.prototype.slice.call(arguments)))
    }
    
    if (this.prototype) {
      fBound.prototype = this.prototype
    } 
	return fBound
  }
}
```
假设我们将调用`bind`的函数称为C，将`fBound`的`prototype`原型对象指向C的`prototype`原型对象（上例中就是`self`），这样的话如果将`fBound`作为构造函数（使用`new`操作符）实例化一个对象，那么这个对象也是C的实例，`this instanceof self`就会返回true。这时就将`self`指向新创建的对象的`this`上就可以达到原生`bind`的效果了（不再固定指定的`this`）。否则，才使用`oThis`，即绑定指定的`this`。

但是这样做会有什么影响？将`fBound`的`prototype`原型对象直接指向`self`的`prototype`原型对象，那么**当修改`fBound`的`prototype`对象时，`self`（上述C函数）的`prototype`对象也会被修改！！**考虑到这个问题，我们需要另外一个`function`来帮我们做个中间人来避免这个问题，我们看看MDN是怎么实现`bind`的。


### MDN提供的Polyfill方案 及 YUI库extend函数实现
MDN针对`bind`没有被广泛支持的兼容性提供了一个实现方案：

```
if (!Function.prototype.bind) {
  Function.prototype.bind = function(oThis) {
    if (typeof this !== 'function') {
      // closest thing possible to the ECMAScript 5
      // internal IsCallable function
      throw new TypeError('Function.prototype.bind - what is trying to be bound is not callable');
    }

    var aArgs = Array.prototype.slice.call(arguments, 1),//这里的arguments是跟oThis一起传进来的实参
      fToBind = this,
      fNOP    = function() {},
      fBound  = function() {
        return fToBind.apply(this instanceof fNOP
          ? this
          : oThis,
          // 获取调用时(fBound)的传参.bind 返回的函数入参往往是这么传递的
          aArgs.concat(Array.prototype.slice.call(arguments)));
      };

    // 维护原型关系
    if (this.prototype) {
      // Function.prototype doesn't have a prototype property
      fNOP.prototype = this.prototype;
    }
    fBound.prototype = new fNOP();

    return fBound;
  };
}
```
发现了吗，和上面经过改造的方案相比，最主要的差异就在于它定义了一个空的`function fNOP`，通过`fNOP`来传递原型对象给`fBound`（通过实例化的方式）。这时，修改`fBound`的`prototype`对象，就不会影响到`self`的`prototype`对象啦～而且`fNOP`是空对象，所以几乎不占内存。

其实这个思路也是YUI库如何实现继承的方法。他的`extend`函数如下：

```
function extend(Child, Parent) {
	var F = function(){};
	F.prototype = Parent.prototype;
	Child.prototype = new F();
	Child.prototype.constructor = Child;
}

```
最后一步是将`Child`的`constructor`指回`Child`。


### 总结
实现一个原生的函数，最重要的是理清楚它的作用和功能，然后逐一去实现它们包括细节，基本上就不会有问题～

这里用到的一些关于**`prototype`**和**`instanceof`**的具体含义，可以参考阮一峰老师的 [prototype 对象](http://javascript.ruanyifeng.com/oop/prototype.html)，相信对你理解JavaScript的原型链和继承会有帮助～

好啦就这样，晚安好梦了各位🌛✨
