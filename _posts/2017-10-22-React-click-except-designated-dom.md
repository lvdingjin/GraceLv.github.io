---
layout: post
title: "React项目场景之：点击document任意节点（除指定区域外）处理状态"
description: ""
category: tech
tags: []
modify: 2017-10-22 15:46
---

update: 2017-10-22


我们常常会遇到这样一个场景：打开一个fix的弹层，需要在用户点击除了它本身以外的任意位置时，关闭弹层。
这样的场景在日常前端的开发中太常见了，这里也不再阐述平常我们用原生js来实现的具体方法，主要讲述下如果是在React项目中遇到这种情况的处理方法。

### [rc-util](https://github.com/react-component/util)
rc-util是一个专为React组件提供一些通用工具的插件，我们可以通过rc-util的Dom模块的一系列函数来操作dom。

1. 引用rc-util中需要用到的模块，以及绑定节点需要用到的findDOMNode方法
  ```
  import addEventBodyListener from 'rc-util/lib/Dom/addEventListener';
  import contains from 'rc-util/lib/Dom/contains';
  import { findDOMNode } from 'react-dom';
  ```
  
2. 通过react的ref绑定需要除外的指定区域的dom节点，方法详见react文档，这里不再赘述。通过一个组件内部的state来控制该组件的显示隐藏。
  ```
  <TimeSelect ref="timemodal" visible={this.state.isShowTimeSelect} />
  ```
  
3. 在组件的componentDidUpdate添加监听事件，监听document元素的点击事件
  ```
  componentDidUpdate() {
    if (this.state.isShowTimeSelect) {
      this.clickOutsideHandle = addEventBodyListener(document, 'mousedown', this.onDocumentClick.bind(this))
      return
    }
  }
  ```
  
  定义document元素的点击事件：
  获取指定区域节点，与当前点击的节点进行比较，如果当前点击的节点不是指定区域的节点，则通过state来隐藏组件，达到点击document任意节点（除指定区域外）关闭弹层的效果。
  ```
  onDocumentClick(e) {
    let timemodal = findDOMNode(this.refs.timemodal)
    if (!contains(timemodal, e.target)) {
      this.setState({
        isShowTimeSelect: false
      })
    }
  }
  ```
  *contains*：检查node是否为root或root的子元素。
  > (root: HTMLElement, node: HTMLElement): boolean
  
  > Check if node is equal to root or in the subtree of root.
  
4. 移除监听事件：在组件的componentWillUnmount和componentDidUpdate都需要移除监听事件
  ```
  componentDidUpdate() {
    if (this.clickOutsideHandle) {
      this.clickOutsideHandle.remove()
      this.clickOutsideHandle = null
    }
  }

  componentWillUnmount() {
    if (this.clickOutsideHandle) {
      this.clickOutsideHandle.remove()
      this.clickOutsideHandle = null
    }
  }
  ```
  
  
以上，完成在React项目中实现点击document任意节点（除指定区域外）隐藏弹层的效果。

