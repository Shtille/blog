---
layout: post
title: "Map enumeration performance comparison"
author: "Shtille"
categories: journal
tags: [code,JavaScript]
---

We need compare two methods of map enumeration:
- via _forEach_ method
- via _entries_ enumeration

Code setup will be:
```js
const N = 100000;
var m = new Map();
for (var i = 0; i < 10; ++i)
	m.set(i,i);
```

The first case:
```js
var x = 0;
m.forEach(function(value){
	x += value;
});
```

The second case:
```js
var x = 0;
for (const [key,value] of m.entries()) {
	x += value;
}
```

According to [benchmark site](https://jsbench.me/) the second one is 58.62% _slower_. This is kinda strange.