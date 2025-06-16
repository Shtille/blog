---
layout: post
title: "Polyline rendering. Part 2"
author: "Shtille"
categories: journal
tags: [C++,GLSL,samples]
image: polyline-rounded.png
---

## Overview

In the [previous article]({{ 'polyline-rendering' | relative_url }}) I created a screen space invariant width polyline rendering.
Now we need to add rounding to segments. The easiest way to achieve that is to discard pixels far than halfwidth radius distance from segment points.

### Coordinates transformation

To calculate that distance we gonna use texture coordinates transformed into desired range:

* `u` coordinate will be transformed into `[-w,l+w]` range;
* `v` coordinate will be transformed into `[-w,+w]` range;

where `l` is length of segment in pixels.

$$
\begin{cases}
u \in [-1,1] \mapsto [-w,l+w] \\
v \in [-1,1] \mapsto [-w,+w]
\end{cases} \tag{1}\label{1}
$$

Let's calculate common case transformation:

$$
x \in [-1,1] \mapsto [a,b]
$$

$$
x{^1/_2}+{^1/_2} \in [0,1]
$$

$$
(x{^1/_2}+{^1/_2})(b-a) \in [0,b-a]
$$

$$
(x{^1/_2}+{^1/_2})(b-a)+a \in [a,b] \tag{2}\label{2}
$$

Finally, we put values from $\eqref{1}$ into $\eqref{2}$:

$$
\begin{cases}
u`=(u{^1/_2}+{^1/_2})(l+w+w)-w \\
v`=u*w
\end{cases} \tag{3}\label{3}
$$

### Shader code

Vertex shader:

```glsl
#version 330 core

layout(location = 0) in vec3 a_prev;
layout(location = 1) in vec3 a_curr;
layout(location = 2) in vec3 a_next;
layout(location = 3) in vec2 a_texcoord;

uniform mat4 u_proj;
uniform mat4 u_view;
uniform mat4 u_model;
uniform vec4 u_viewport;
uniform float u_width;

noperspective out vec2 v_texcoord;
flat out float v_length;
flat out float v_radius;

vec2 project(vec4 clip)
{
    vec3 ndc = clip.xyz / clip.w;
    vec2 screen = (ndc.xy * 0.5 + vec2(0.5)) * u_viewport.zw + u_viewport.xy;
    return screen;
}
vec4 unproject(vec2 screen, float z, float w)
{
    vec2 ndc = ((screen - u_viewport.xy) / u_viewport.zw - vec2(0.5)) * 2.0;
    return vec4(ndc.x * w, ndc.y * w, z, w);
}

void main()
{
    mat4 mvp = u_proj * u_view * u_model;

    vec4 clip_curr = mvp * vec4(a_curr, 1.0);
    vec4 clip_prev = mvp * vec4(a_prev, 1.0);
    vec4 clip_next = mvp * vec4(a_next, 1.0);

    vec2 curr = project(clip_curr);
    vec2 prev = project(clip_prev);
    vec2 next = project(clip_next);

    vec2 direction = next - prev;
    v_length = length(direction);
    direction /= v_length; // normalize
    vec2 normal = vec2(-direction.y, direction.x);

    float w = u_width * 0.5;
    vec2 offset_x = direction * (a_texcoord.x * w);
    vec2 offset_y = normal    * (a_texcoord.y * w);
    vec2 screen = curr + offset_x + offset_y;

    gl_Position = unproject(screen, clip_curr.z, clip_curr.w);

    v_texcoord = a_texcoord;
    v_radius = w;
}
```

Fragment shader:

```glsl
#version 330 core

uniform vec4 u_color;

out vec4 color;
noperspective in vec2 v_texcoord;
flat in float v_length;
flat in float v_radius;

void main()
{
    vec2 texcoord = v_texcoord;
    float length = v_length;
    float w = v_radius;
    
    vec2 position;
    position.x = (texcoord.x * 0.5 + 0.5) * (length + w + w) - w; // [-w, l+w]
    position.y = texcoord.y * w; // [-w, w]

    if (position.x <= 0.0)
    {
        float dist = length(position);
        if (dist > w)
            discard;
    }
    else if (position.x >= length)
    {
        float dist = length(vec2(position.x - length, position.y));
        if (dist > w)
            discard;
    }
    color = u_color;
}
```

## Conclusion

<img src="{{ '/assets/img/polyline-rounded.png' | relative_url }}">

We achieved polyline rendering with rounded joins.
Polyline rendering with different caps on sides will be covered in the next post.