---
title: js中对象assign的用法
date: 2018-03-03 16:50:15
tags: js
categories: 技术
---

ES6新的方法Object.assign()
Object.assign方法用于对象的合并，将源对象（ source ）的所有可枚举属性，复制到目标对象（ target ）

```js
var target = { a: 1 };  
var source1 = { b: 2 };  
var source2 = { c: 3 };  
Object.assign(target, source1, source2);  
target // {a:1, b:2, c:3}
```

