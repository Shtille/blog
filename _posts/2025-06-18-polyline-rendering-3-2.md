---
layout: post
title: "Polyline rendering. Part 3.2"
author: "Shtille"
categories: journal
tags: [C++,GLSL,samples]
image: polyline-caps.png
---

## Overview

In the [previous post]({{ 'polyline-rendering-3' | relative_url }}) I created a screen space invariant width polyline rendering with different caps. But there's an optimization available.

## Optimization

No need to store offset mask in vertex, we could store point type itself $T$ and compute the offset mask value $M$.
Let's define point type $T$ as:

$$
T =
\begin{cases}
-1\quad\text{if begin point}
1\quad\text{if end point}
0\quad\text{if join point}
\end{cases} \tag{1}\label{1}
$$

Then offset mask value $M$ will be:

$$
M = 1-\left|T\right|
$$

This also won't require additional attributes.

### Vertex layout

Vertex layout is following:

```cpp
struct alignas(4) Vertex
{
    Point3DF position; // vec3
    PointF texcoord;   // vec2
    float pointType;   // -1 begin, 1 end, 0 other
};
```

### Data creation code

Partial data creation code is following:

```cpp
// Point types (determines begin and end of polyline)
n = 0;
// First segment points
vertices[n++].pointType = -1.0f;
vertices[n++].pointType = -1.0f;
vertices[n++].pointType = -1.0f;
vertices[n++].pointType = -1.0f;
for (uint32_t i = 0; i < numSegments; ++i) // enumerate original segments
{
    const bool isBegin = i == 0;
    const bool isEnd   = i == numSegments - 1;
    const float beginType = isBegin ? -1.0f : 0.0f;
    const float endType   =   isEnd ?  1.0f : 0.0f;
    vertices[n++].pointType = beginType;
    vertices[n++].pointType = beginType;
    vertices[n++].pointType = endType;
    vertices[n++].pointType = endType;
}
// Last segment points
vertices[n++].pointType = 1.0f;
vertices[n++].pointType = 1.0f;
vertices[n++].pointType = 1.0f;
vertices[n++].pointType = 1.0f;
```

### Attributes layout

The attribute layout changes are following:

```cpp
const uint8_t* pointTypeOffset = texcoordOffset + sizeof(PointF);
funcs->glVertexAttribPointer(4, 1, GL_FLOAT, GL_FALSE, stride, pointTypeOffset); // float a_point_type
funcs->glEnableVertexAttribArray(4);
```

### Shader code

Vertex shader:

```glsl
#version 330 core

layout(location = 0) in vec3 a_position_prev;
layout(location = 1) in vec3 a_position_curr;
layout(location = 2) in vec3 a_position_next;
layout(location = 3) in vec2 a_texcoord;
layout(location = 4) in float a_point_type;

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

    vec2 direction = next - prev;
    v_length = length(direction);
    direction /= v_length; // normalize
    vec2 normal = vec2(-direction.y, direction.x);

    float w = u_width * 0.5;
    float offset_x = a_texcoord.x * w;
    float offset_y = a_texcoord.y * w;

    if (u_cap_style == 1) // flat
        offset_x *= 1.0 - abs(a_point_type); // 0 for begin or end, and 1 otherwise

    vec2 direction_offset = direction * offset_x;
    vec2 normal_offset    = normal    * offset_y;
    vec2 screen = curr + direction_offset + normal_offset;

    gl_Position = unproject(screen, clip_curr.z, clip_curr.w);

    v_texcoord = a_texcoord;
    v_radius = w;
    v_point_type = a_point_type;
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

We optimized polyline rendering with different caps.
Polyline rendering with different join styles will be covered in the next post.