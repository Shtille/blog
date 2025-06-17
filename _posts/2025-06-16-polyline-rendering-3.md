---
layout: post
title: "Polyline rendering. Part 3"
author: "Shtille"
categories: journal
tags: [C++,GLSL,samples]
image: polyline-caps.png
---

## Overview

In the [previous post]({{ 'polyline-rendering-2' | relative_url }}) I created a screen space invariant width polyline rendering with rounded joins. Now I gonna cover different caps on sides.

## Implementation

Caps can have different styles.

### Cap styles

The cap style defines how the end points of lines are drawn.
We will base cap styles on [Qt version](https://doc.qt.io/qt-6/qpen.html#cap-style). So there will be following styles:

* square
* flat
* round

#### Square cap style

A square cap style is a square line end that covers the end point and extends beyond it by half the line width.
This style has already been partly implemented in the [first post]({{ 'polyline-rendering' | relative_url }}).

#### Flat cap style

A flat cap style is a square line end that does not cover the end point of the line. The same as square, but without offset in `u` direction.

#### Round cap style

A round cap style is a rounded line end covering the end point. This style has been fully implemented in [previous post]({{ 'polyline-rendering-2' | relative_url }}).

### Line end determinition

We need to determine line end. The obvious way is to have an additional vertex attribute for offset mask. Mask is equal to zero on start and end of polyline, and equal to one on joins.

$$
M = \begin{cases} 0 \cr 1 \end{cases}
$$

Masks layout will be following:

$$
\begin{cases}
(0, 0, 0, 0) \\
(0, 0, 1, 1) \\
(1, 1, 1, 1) \\
\dotso \\
(1, 1, 1, 1) \\
(1, 1, 0, 0) \\
(0, 0, 0, 0)
\end{cases} \tag{1}\label{1}
$$

To determine the point type (begin, end, common) we gonna use two additional attributes for mask like we did for segment direction calculation and get mask direction vector for segment:

$$
d_m = (M_{next} - M_{curr}) + (M_{curr} - M_{prev}) = M_{next} - M_{prev}
$$

Mask difference layout will be:

$$
\begin{cases}
(0, 0, 0, 0) \\
(1, 1, 1, 1) \\
(0, 0, 0, 0) \\
\dotso \\
(0, 0, 0, 0) \\
(-1, -1, -1, -1) \\
(0, 0, 0, 0)
\end{cases} \tag{2}\label{2}
$$

So if as we want to have -1 for begin, 1 for end and 0 otherwise, point type could be computed as:

$$
T_p = -d_m (1-M) = (M_{next} - M_{prev})(M - 1) \tag{3}\label{3}
$$

### Vertex layout

Vertex layout is following:

```cpp
struct alignas(4) Vertex
{
    Point3DF position; // vec3
    PointF texcoord;   // vec2
    float mask;        // float
};
```

### Data creation code

Partial data creation code is following:

```cpp
// Masks (determines begin and end of polyline)
n = 0;
// First segment points
vertices[n++].mask = 0.0f;
vertices[n++].mask = 0.0f;
vertices[n++].mask = 0.0f;
vertices[n++].mask = 0.0f;
for (uint32_t i = 0; i < numSegments; ++i) // enumerate original segments
{
    const bool isBegin = i == 0;
    const bool isEnd = i == numSegments - 1;
    const float beginMask = isBegin ? 0.0f : 1.0f;
    const float endMask = isEnd ? 0.0f : 1.0f;
    vertices[n++].mask = beginMask;
    vertices[n++].mask = beginMask;
    vertices[n++].mask = endMask;
    vertices[n++].mask = endMask;
}
// Last segment points
vertices[n++].mask = 0.0f;
vertices[n++].mask = 0.0f;
vertices[n++].mask = 0.0f;
vertices[n++].mask = 0.0f;
```

### Attributes layout

The attribute layout changes are following:

```cpp
const uint8_t* maskCurrOffset = texcoordOffset + sizeof(PointF);
const uint8_t* maskPrevOffset = maskCurrOffset - 2*stride;
const uint8_t* maskNextOffset = maskCurrOffset + 2*stride;
funcs->glVertexAttribPointer(4, 1, GL_FLOAT, GL_FALSE, stride, maskPrevOffset); // float a_mask_prev
funcs->glEnableVertexAttribArray(4);
funcs->glVertexAttribPointer(5, 1, GL_FLOAT, GL_FALSE, stride, maskCurrOffset); // float a_mask_curr
funcs->glEnableVertexAttribArray(5);
funcs->glVertexAttribPointer(6, 1, GL_FLOAT, GL_FALSE, stride, maskNextOffset); // float a_mask_next
funcs->glEnableVertexAttribArray(6);
```

### Cap style definition

Cap style would be defines as following enum:

```cpp
enum class CapStyle
{
    Square,
    Flat,
    Round
};
```

### Shader code

Vertex shader:

```glsl
#version 330 core

layout(location = 0) in vec3 a_position_prev;
layout(location = 1) in vec3 a_position_curr;
layout(location = 2) in vec3 a_position_next;
layout(location = 3) in vec2 a_texcoord;
layout(location = 4) in float a_mask_prev;
layout(location = 5) in float a_mask_curr;
layout(location = 6) in float a_mask_next;

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
        offset_x *= a_mask_curr;

    vec2 direction_offset = direction * offset_x;
    vec2 normal_offset    = normal    * offset_y;
    vec2 screen = curr + direction_offset + normal_offset;

    gl_Position = unproject(screen, clip_curr.z, clip_curr.w);

    v_texcoord = a_texcoord;
    v_radius = w;
    v_point_type = (a_mask_next - a_mask_prev)*(a_mask_curr - 1.0);
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
noperspective in float v_point_type; // -1, 0, 1

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

We achieved polyline rendering with different caps.
Polyline rendering with different join styles will be covered in the next post.