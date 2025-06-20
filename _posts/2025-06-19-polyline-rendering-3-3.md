---
layout: post
title: "Polyline rendering. Part 3.3"
author: "Shtille"
categories: journal
tags: [C++,GLSL,samples]
image: polyline-caps.png
---

## Overview

In the [previous post]({{ 'polyline-rendering-3-2' | relative_url }}) I optimized polyline rendering with different caps. But we need to prepare our code for join implementation.

## Implementation

Instead of usage of semi previous and semi next points for direction calculation we could use real previous and next points, and calculate current segment direction vector via linear interpolation of two directions.

Lets assume we have point $P_i$, so direction vectors would be:

$$
\begin{cases}
\mathbf{d}_{i} = \mathbf{P}_{i} - \mathbf{P}_{i-1} \\
\mathbf{d}_{i+1} = \mathbf{P}_{i+1} - \mathbf{P}_{i}
\end{cases} \tag{1}\label{1}
$$

This point can lean on any of adjacent segments. To determine the correct segment we could use *x* value of texture coordinate:

$$
x \in [-1,1]
$$

Transform it into unit range:

$$
t = x{^1/_2}+{^1/_2}
$$

So the current vertex point direction $d_{vi}$ will be:

$$
\mathbf{d}_{vi} = \mathbf{d}_{i} t + \mathbf{d}_{i+1} (1-t) \tag{2}\label{2}
$$

### Attributes layout

The attribute layout becomes following:

```cpp
// Attributes layout
const GLsizei stride = sizeof(Vertex);
const uint8_t* base = nullptr;
const uint8_t* prevOffset = base;
const uint8_t* currOffset = prevOffset + 4*stride; // offset to start of actual current data
const uint8_t* nextOffset = currOffset + 4*stride;
const uint8_t* texcoordOffset = currOffset + sizeof(Point3DF);
funcs->glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, stride, prevOffset); // vec3 a_position_prev
funcs->glEnableVertexAttribArray(0);
funcs->glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, stride, currOffset); // vec3 a_position_curr
funcs->glEnableVertexAttribArray(1);
funcs->glVertexAttribPointer(2, 3, GL_FLOAT, GL_FALSE, stride, nextOffset); // vec3 a_position_next
funcs->glEnableVertexAttribArray(2);
funcs->glVertexAttribPointer(3, 3, GL_FLOAT, GL_FALSE, stride, texcoordOffset); // vec2 a_texcoord
funcs->glEnableVertexAttribArray(3);
```

### Shader code

Vertex shader:

```glsl
#version 330 core

layout(location = 0) in vec3 a_position_prev;
layout(location = 1) in vec3 a_position_curr;
layout(location = 2) in vec3 a_position_next;
layout(location = 3) in vec3 a_texcoord;

uniform mat4 u_proj;
uniform mat4 u_view;
uniform mat4 u_model;
uniform vec4 u_viewport;
uniform float u_width;
uniform int u_cap_style;

noperspective out vec2 v_texcoord;
flat out float v_length;
flat out float v_radius;
noperspective out float v_point_type;

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

    vec4 clip_curr = mvp * vec4(a_position_curr, 1.0);
    vec4 clip_prev = mvp * vec4(a_position_prev, 1.0);
    vec4 clip_next = mvp * vec4(a_position_next, 1.0);

    vec2 curr = project(clip_curr);
    vec2 prev = project(clip_prev);
    vec2 next = project(clip_next);

    vec2 tangent1 = curr - prev;
    vec2 tangent2 = next - curr;

    vec2 direction = mix(tangent2, tangent1, a_texcoord.x * 0.5 + 0.5);
    v_length = length(direction);
    direction /= v_length; // normalize
    vec2 normal = vec2(-direction.y, direction.x);

    float w = u_width * 0.5;
    float offset_x = a_texcoord.x * w;
    float offset_y = a_texcoord.y * w;

    v_point_type = a_texcoord.z;

    if (u_cap_style == 1) // flat
        offset_x *= 1.0 - abs(v_point_type); // 0 for begin or end, and 1 otherwise

    vec2 direction_offset = direction * offset_x;
    vec2 normal_offset    = normal    * offset_y;
    vec2 screen = curr + direction_offset + normal_offset;

    gl_Position = unproject(screen, clip_curr.z, clip_curr.w);

    v_texcoord = a_texcoord.xy;
    v_radius = w;
}
```

Fragment shader:

```glsl
#version 330 core

uniform int u_cap_style;
uniform vec4 u_color;

out vec4 color;
noperspective in vec2 v_texcoord;
flat in float v_length;
flat in float v_radius;
noperspective in float v_point_type;

void main()
{
    vec2 texcoord = v_texcoord;
    float length = v_length;
    float w = v_radius;
    bool need_discard = u_cap_style == 2; // round
    
    vec2 position = vec2(texcoord.x * 0.5 + 0.5, texcoord.y * w);

    if (v_point_type < -0.5) // == -1, begin, [0, l+w]
    {
        position.x = position.x * (length + w);
    }
    else if (v_point_type > 0.5) // == 1, end, [-w, l]
    {
        position.x = position.x * (length + w) - w;
    }
    else // == 0, common, [-w, l+w]
    {
        position.x = position.x * (length + w + w) - w;
        need_discard = true;
    }

    if (need_discard && position.x < 0.0)
    {
        float dist = length(position);
        if (dist > w)
            discard;
    }
    else if (need_discard && position.x > length)
    {
        float dist = length(vec2(position.x - length, position.y));
        if (dist > w)
            discard;
    }
    color = u_color;
}
```

## Conclusion

<img src="{{ '/assets/img/polyline-caps.png' | relative_url }}">

We prepared our code for polyline rendering with different join styles implementation.
It will be covered in the next post.