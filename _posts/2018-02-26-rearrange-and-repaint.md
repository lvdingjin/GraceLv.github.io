---
layout: post
title: "重排、重绘和优化他们的方案"
description: ""
category: tech
tags: []
modify: 2018-02-26 17:52:34
---

update: 2018-02-26


2018年第一篇文章，决定总结下浏览器的重排与重绘以及与之相关的一些优化点和影响~
其实是对很早以前自己遇到的一些场景和当时记录的笔记的重新整理啦&#x1F61D;&#x1F61D;&#x1F61D;

## 重排和重绘
理论上来说，浏览器下载完成页面中的所有组件（HTML标记、javascript、CSS、图片）之后会解析并生成两个内部数据结构：**DOM树和渲染树**。DOM树表示页面结构，渲染树表示DOM节点如何显示。

那么重排和重绘又是什么东东呢？顾名思义，**重排**就是重新排列，而**重绘**就是重新绘制。

那他们分别在什么时候发生呢？
> 当页面布局和元素的几何属性（宽和高）发生改变时，浏览器就会发生“重排”。完成重排后，浏览器会重新绘制受影响的部分到屏幕中，即发生“重绘”。


具体列举一些发生重排的场景：

* 页面渲染器初始化
* 添加或删除**可见**的DOM元素
* 元素**位置**改变
* 元素**尺寸**改变（包括：margin、padding、border-width、width、height等属性改变）
* **内容**改变（比如：文本改变导致行数改变、图片被另一个不同尺寸的图片替代等）
* 浏览器窗口尺寸改变

还有一些改变，会导致整个页面的重排，比如：当滚动条出现时。

## 渲染树变化的队列与刷新
由于每次重排都会产生计算的消耗，所以大多数浏览器通过**`队列化修改`**并**`批量执行`**来优化重排过程。

而**获取布局信息的操作**，则会导致这个队列的刷新，比如以下方法：

* offsetTop, offsetLeft, offsetWidth, offsetHeight
* scrollTop, scrollLeft, scrollWidth, scrollHeight
* clientTop, clientLeft, clientWidth, clientHeight
* getComputedStyle() (IE下是currentStyle)

这些属性和方法需要返回**最新的布局信息**，因此浏览器不得不执行渲染队列中的“待处理变化”并触发重排以返回正确的值。

## 性能优化，最小化重排和重绘
重排和重绘的发生，无疑是影响页面渲染性能的一大原因。因此如何最小化重排和重绘就变得非常有必要了。

总的来说，可以分为以下几种方式：

1. 合并多次对DOM和样式的修改，然后一次处理掉。
2. 缓存布局信息。
3. 让元素脱离动画流。

### 1、合并多次对DOM和样式的修改

#### 改变样式
例：

```
let el = document.getElementById(‘mydiv’);
el.style.borderLeft = ‘1px’;
el.style.borderRight = ‘2px’;
el.style.padding = ‘5px’;
```

这种方式在最糟糕的情况下会导致浏览器触发三次重排。

优化方案：

```
let el = document.getElementById(‘mydiv’);
el.style.cssText = ‘border-left:1px;border-right:2px;padding:5px;’;
//如果想保留现有样式，可以把它附加在cssText字符串后面：el.style.cssText += ‘;border-left:1px;’;
```

另一种一次性修改样式的办法是修改CSS的class名称，尽管它可能会带来轻微的性能影响，因为改变类时需要检查级联样式。

#### 批量修改DOM
当需要对DOM元素进行一系列操作时，可以通过以下步骤减少重绘和重排的次数：

**1. 使元素脱离文档流（此操作会触发一次重排）**

**2. 对其应用多重改变**

**3. 把元素带回文档中（触发第二次重排）**


使元素脱离文档流，方法有很多：

* 隐藏元素，应用修改，然后重新显示元素
* 使用文档片段在当前DOM之外构建一个子树，再把它拷贝回文档
* 将原始元素拷贝到一个脱离文档的节点中，修改副本，完成后再替换原始元素

那么哪种方式更好呢？

假设`appendDataToElement`是一个封装好的用来更新指定节点数据的通用函数，第一个参数是指定节点，第二个参数是要更新的数据。

上述方案一就是：

```
let ul = document.getElementById(‘mylist’);
ul.style.display = ‘none’;
appendDataToElement(ul, data);
ul.style.display = ‘block’;
```

上述方案二就是：

```
let fragment = document.createDocumentFragment();
appendDataToElement(fragment, data);
document.getElementById(‘mylist’).appendChild(fragment);
```

可以看到，这种方式只触发一次重排，并且只访问了一次实时的DOM。

上述方案三就是：

```
let old = document.getElementById(‘mylist’);
let clone = old.cloneNode(true);
appendDataToElement(clone, data);
old.parentNode.replaceChild(clone, old);
```

这种方式为需要修改的节点创建一个备份，然后对副本进行操作，操作完成以后再用新的节点代替旧的节点。

综合比较，**你会发现第二种方式是最好的，为啥呢？因为他们所产生的DOM遍历和重排次数是最少的**。


### 2、缓存布局信息
减少布局信息的获取次数，获取后把它赋值给局部变量，然后再操作局部变量。

### 3、让元素脱离动画流

1. 使用绝对定位来布局页面上的动画元素，将它脱离文档流。
2. 让元素动起来。当它扩大时，会临时覆盖部分页面，但这只是页面一个小区域的重绘过程，不会产生重排并重绘页面的大部分内容。
3. 当动画结束时恢复定位，从而只会下移一次文档的其他元素。


重排和重绘是一个非常耗性能的行为，在开发的过程中，应该要养成下意识去使用更优写法的习惯，而这就需要慢慢的积累方法以及不怕麻烦的实践啦&#x1F609;~本文列举的还只是JavaScript中比较常见的场景而已，欢迎补充更多的优化方案，感谢感恩！新年快乐，2018一起加油&#x270A;&#x270A;&#x270A;
