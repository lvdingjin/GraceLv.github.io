---
layout: post
title: "正则表达式使用全局检索“g”标志引起的问题"
description: ""
category: tech
tags: []
modify: 2017-11-07 11:42:20
---

update: 2017-11-07


正则表达式在日常开发中简直可以说是无处不在，功能之强大可以用于检查任何字符串。
最近遇到一个问题，研究后才发现关于**全局检索“g”标志**的套路还是很深的，需要将规则熟记于心、严谨慎用哦~

先描述下问题场景：一个搜索输入框，对用户输入的关键词在固有的列表中进行匹配，返回匹配成功的项目列表。

搜索函数代码：

```
let handleSearch = (value, arr) => {
	value = (value + '').replace(/([.?*+^$[\]\\(){}|-])/g, '\\$1');
	let pattern = new RegExp(`${value}`, 'g');
	let result = [];
	arr && arr.map(item => {
	  if (pattern.test(item.name)) {
	    result.push(item);
	  }
	});
	return result;
}
```

当我的固有列表中，有两项**连续**的能匹配到的Object时，却出现了只匹配到第一项的结果：

```
handleSearch('library', [{id: 1, name: 'library'}, {id: 2, name: 'library'}, {id: 1, name: 'bank'}]);
```

输出：
```[{id: 1, name: 'library'}]```

这显然不是我们期望的结果。

这时候可以尝试下，在搜索函数的循环体内，打印一下正则的lastIndex看看。
```
arr && arr.map(item => {
  console.log(pattern.lastIndex);
  if (pattern.test(item.name)) {
    result.push(item);
  }
});
```

发现第一次lastIndex为0，第二次为7，第三次又为0。

这就是导致结果不对的主要原因。

### RegExp.lastIndex ###
> lastIndex 是正则表达式的一个可读可写的整型属性，用来指定下一次匹配的起始索引。

**但是这个索引，只有正则表达式使用了表示全局检索“g”标志时，才会生效。lastIndex有一套规则：**
> 如果 lastIndex 大于字符串的长度，则 regexp.test 和 regexp.exec 将会匹配失败，然后 lastIndex 被设置为 0。
> 
> 如果 lastIndex 等于字符串的长度，且该正则表达式匹配空字符串，则该正则表达式匹配从 lastIndex 开始的字符串。
> 
> 如果 lastIndex 等于字符串的长度，且该正则表达式不匹配空字符串 ，则该正则表达式不匹配字符串，lastIndex 被设置为 0。
> 
> 否则，lastIndex 被设置为紧随最近一次成功匹配的下一个位置。

也就是说，在上面那个例子中，第一次进入循环，匹配成功后，lastIndex即刻变为了匹配成功的下一个位置7，第二次进入循环时，因为起始索引为7，满足上述规则第3条，不匹配字符串，同时lastIndex变为了0。

如何解决呢？

1. **将构造函数放到循环体内，保证每一次验证的都是新的实例对象；**
2. **每一次匹配成功后，都手动将lastIndex设为0**（这样做其实就相当于可以不用设置全局标志g了，只匹配一次就可以了）。

另外，其实这个例子可以不使用“g”，这个要依据使用场景而定。在必要使用“g”的情况下，请注意以上规则会带来的影响，及时规避问题呦~
