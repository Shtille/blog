---
layout: post
title: "Replace map keys function"
author: "Shtille"
categories: journal
tags: [code,JavaScript]
---

I've made a function to replace _Map_ keys with specific pattern defined by another _Map_ class.

```js
/**
 * Replaces keys with specific pattern.
 * 
 * @param {Map} map  The replace map. Contains (old key, new key) pairs.
 */
Map.prototype.replaceKeys = function(map) {
	if (map.size == 0)
		return;
	let newMap = new Map();
	// At first add changed items
	map.forEach(function(newKey, oldKey){
		if (this.has(oldKey)) {
			newMap.set(newKey, this.get(oldKey));
		}
	}, this);
	// Then add remained ones
	this.forEach(function(value, key){
		if (!map.has(key)) {
			newMap.set(key, value);
		}
	}, this);
	// Finally replace the content
	this.clear();
	newMap.forEach(function(value, key){
		this.set(key, value);
	}, this);
};
```