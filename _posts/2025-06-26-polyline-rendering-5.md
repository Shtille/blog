---
layout: post
title: "Polyline rendering. Part 5"
author: "Shtille"
categories: journal
tags: [C++,GLSL,samples]
image: polyline-dashed.png
---

## Overview

In the [previous post]({{ 'polyline-rendering-4' | relative_url }}) I implemented version with diffent joins. Now I gonna cover polyline rendering with dash pattern.

### Dash line classification

Let's classify dash patterns.

#### Rounded caps presence

At first, dash pattern can be differentiated on rounded cap presence flag:

1. Without rounded caps
2. With rounded caps
3. Mixed

First type is very straightforward: we just discard pixels that lie inside holes.
At the same time second type allows us to draw dotted pattern.

#### By phase usage

Next, dash pattern can be differentiated on the pattern offset (phase):

1. With offset
2. Without offset

The offset allows us to get a nice looking pattern along the entire length of the line.

#### By distance type

At last, dash pattern can be differentiated on distance type:

1. Global space distance
2. Screen space distance

Global space distance is more preferable for 3D polylines.

## Implementation

Let's implement screen space based dash line without rounded caps.

We need a texture to hold our pattern data:

$$
(1,1,1,1,0,0,1,1,0)
$$

where $1$'s discribe solid line parts, and $0$'s discribe holes.

This data can be packed as following array:

$$
(4,2,2,1)
$$

where even indices discribe solid parts, odd indices - holes, and each value contains number of parts.

Now let's introduce the following notations:

* $d_\tau$ - period distance
* $d_\phi$ - pattern phase
* $d_u$ - size of a part (unit size)
* $n$ - width of texture (number of units)

$$
d_\tau = n d_u \tag{1}\label{1}
$$

Then index in array of texture data $k$ for certain distance value $d$ can be obtained as:

$$
k = ((d + d_\tau - d_\phi) \mod d_\tau)/d_u \tag{2}\label{2}
$$

As we use texture coordinate $u$ in range $[0,1]$ we can calculate it from equations $\eqref{1}$ and $\eqref{2}$ for our special case:

$$
u = k/n
$$

$$
u = (d \mod d_\tau)/d_\tau
$$

### Data creation code

Partial data creation code is following:

```cpp
	// Texture data
	// ------------
	const std::vector<uint32_t> pattern = {2,1,1,1};
	texture_width_ = 0;
	for (uint32_t value : pattern)
		texture_width_ += value;

	n = 0;
	texture_data_ = new uint8_t[texture_width_];
	for (uint32_t i = 0; i < pattern.size(); ++i)
	{
		uint8_t value = (i % 2) == 0 ? 255 : 0;
		uint8_t count = pattern[i];
		while (count--)
			texture_data_[n++] = value;
	}
```

### Shader code

Fragment shader partial code:

```glsl
uniform float u_dash_period;
uniform sampler1D u_texture;
// ....
// Transform pixel distance to range [0,1]
float texcoord_x = mod(position.x, u_dash_period) / u_dash_period;
float value = texture(u_texture, texcoord_x).x;
if (value < 0.5)
{
	discard;
}
```

## Conclusion

<img src="{{ '/assets/img/polyline-dashed.png' | relative_url }}">

We achieved polyline rendering with dash pattern.

If we'd decided to use dash phase (offset), it wouldn't be easy to implement, since screen space distance usage require to store accumulated distance value in shader. Version with global space distance would require additional precalculated per-vertex distance attributes.