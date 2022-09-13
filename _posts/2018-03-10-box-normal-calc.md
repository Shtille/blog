---
layout: post
title: "Box normal calculation"
author: "Shtille"
categories: journal
tags: [C++,GLSL,samples]
#image: cards.jpg
---

In ray tracing we need to calculate box normal from ray intersection. Here's the solution.

```cpp
vec3 BoxNormal(const Box box, const vec3 point)
{
	vec3 center = (box.max + box.min) * 0.5;
	vec3 size = (box.max - box.min) * 0.5;
	vec3 pc = point - center;
	// step(edge,x) : x < edge ? 0 : 1
	vec3 normal = vec3(0.0);
	normal += vec3(sign(pc.x), 0.0, 0.0) * step(abs(abs(pc.x) - size.x), kEpsilon);
	normal += vec3(0.0, sign(pc.y), 0.0) * step(abs(abs(pc.y) - size.y), kEpsilon);
	normal += vec3(0.0, 0.0, sign(pc.z)) * step(abs(abs(pc.z) - size.z), kEpsilon);
	return normalize(normal);
}
```