---
layout: post
title: "Map enumeration performance comparison"
author: "Shtille"
categories: journal
tags: [code,JavaScript]
---

We need compare two methods of map enumeration:
- via _forEach_ method
- via _keys_

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
var key,value;
for (key in m) {
	value = m[key];
	x += value;
}
```

According to [benchmark site](https://jsbench.me/) the first one is 94.99% _slower_. So the second one is ~20 times faster.