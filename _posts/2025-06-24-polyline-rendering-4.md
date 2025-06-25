---
layout: post
title: "Polyline rendering. Part 4"
author: "Shtille"
categories: journal
tags: [C++,GLSL,samples]
image: polyline-joins.png
---

## Overview

In the [previous post]({{ 'polyline-rendering-3-3' | relative_url }}) I prepared the code for join styles implementation. But after [geometry shader performance testing]({{ 'polyline-performance' | relative_url }}) I decided to make a version with geometry shader.

## Implementation

### Join styles

There are three join styles:

* bevel
* miter
* round

#### Bevel join style

The bevel join style fills the triangular notch between the two lines.

#### Miter join style

The miter join style extends the lines to meet at an angle.

#### Round join style

The round join style fills a circular arc between the two lines. This style has been implemented earlier.

### Miter point calculation

To calculate miter point we need to know a direction before the join and a direction after.

<img src="{{ '/assets/img/polyline-calc-join.png' | relative_url }}">

In the provided figure we have join point $P_1$, previous point $P_0$ and next point $P_2$. Miter point $M_0$ has it's opposite point $M_1$ of quads intersection.

* Points $A_0$ and $A_1$ are derived from first segment point $P_0$.
* Points $B_0$ and $B_1$ are derived from first segment point $P_1$.
* Points $C_0$ and $C_1$ are derived from second segment point $P_1$.
* Points $D_0$ and $D_1$ are derived from second segment point $P_2$.

Let's define segments directions:

$$
\begin{cases}
d_1 = P_1 - P_0 \\
d_2 = P_2 - P_1
\end{cases} \tag{1}\label{1}
$$

and angles:

$$
\begin{cases}
\alpha = \widehat{B_0 M_0 C_0} \\
\beta = \frac \alpha 2
\end{cases} \tag{2}\label{2}
$$

Angle between directions:

$$
\cos{\alpha} = - \vec{d_1} \cdot \vec{d_2}
$$

For quad halfwidth $w$ distance to miter point $l$:

$$
l = B_0 M_0 = \frac {w} {\tan{\beta}}
$$

$$
l = w \frac {\cos{\beta}} {\sin{\beta}}
$$

$$
l = w \sqrt{\frac{1+\cos{\alpha}}{1-\cos{\alpha}}}
$$

And $M_0$ will be computed as:

$$
M_0 = B_0 + \vec{d_1} l
$$

#### Restrictions

Miter point can't be calculated for angle $\alpha$ close to zero. Also we should define miter limit $l_{\max}$.
So there will be following restrictions:

$$
\begin{cases}
l \le l_{\max} \\
\alpha \ge \alpha_\min
\end{cases} \tag{3}\label{3}
$$

## Code

### Application code

#### Data creation

```cpp
bool PolylineDrawer::CreateData(const PointArray& points)
{
	uint32_t num_points = static_cast<uint32_t>(points.size());
	if (num_points < 2) return false;

	// We add one point before and one point after to have access to previous and next segments.
	num_vertices_ = num_points + 2;
	vertices_array_ = new uint8_t[num_vertices_ * sizeof(Vertex)];
	Vertex* vertices = reinterpret_cast<Vertex*>(vertices_array_);

	// Position
	uint32_t n = 0;
	// First point is extrapolated one from the first segment.
	Point first_point;
	first_point[0] = points[0][0] + (points[0][0] - points[1][0]);
	first_point[1] = points[0][1] + (points[0][1] - points[1][1]);
	vertices[n++].position = first_point;
	for (uint32_t i = 0; i < num_points; ++i)
	{
		const Point& point = points[i];
		vertices[n++].position = {point[0], point[1]};
	}
	// The last point is extrapolated one from the last segment.
	Point last_point;
	last_point[0] = points[num_points-1][0] + (points[num_points-1][0] - points[num_points-2][0]);
	last_point[1] = points[num_points-1][1] + (points[num_points-1][1] - points[num_points-2][1]);
	vertices[n++].position = last_point;

	// Point type
	n = 0;
	vertices[n++].point_type = 0.0f; // value here doesn't matter
	for (uint32_t i = 0; i < num_points; ++i)
	{
		vertices[n++].point_type = (i == 0) ? -1.0f : (i == num_points - 1) ? 1.0f : 0.0f;
	}
	vertices[n++].point_type = 0.0f; // value here doesn't matter

	return true;
}
```

#### Attributes layout

```cpp
	const GLsizei stride = sizeof(Vertex);
	const uint8_t* base = nullptr;
	const uint8_t* prev_offset = base;
	const uint8_t* curr_offset = prev_offset + stride;
	const uint8_t* next_offset = curr_offset + stride;
	const uint8_t* point_type_curr_offset = curr_offset + sizeof(Point);
	const uint8_t* point_type_next_offset = point_type_curr_offset + stride;
	glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, stride, prev_offset); // vec2 a_position_prev
	glEnableVertexAttribArray(0);
	glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, stride, curr_offset); // vec2 a_position_curr
	glEnableVertexAttribArray(1);
	glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, stride, next_offset); // vec2 a_position_next
	glEnableVertexAttribArray(2);
	glVertexAttribPointer(3, 1, GL_FLOAT, GL_FALSE, stride, point_type_curr_offset); // float a_point_type_curr
	glEnableVertexAttribArray(3);
	glVertexAttribPointer(4, 1, GL_FLOAT, GL_FALSE, stride, point_type_next_offset); // float a_point_type_next
	glEnableVertexAttribArray(4);
```

#### Rendering

```cpp
void PolylineDrawer::Render()
{
	ActivateShader();

	glBindVertexArray(vertex_array_object_);
	glDrawArrays(GL_POINTS, 0, num_vertices_ - 3);
	glBindVertexArray(0);

	DeactivateShader();
}
```

### Shader code

#### Vertex shader

```glsl
#version 330 core

layout (location = 0) in vec2 a_position_prev;
layout (location = 1) in vec2 a_position_curr;
layout (location = 2) in vec2 a_position_next;
layout (location = 3) in float a_point_type_curr;
layout (location = 4) in float a_point_type_next;

out VS_OUT {
	vec4 position_prev;
	vec4 position_next;
	float point_type_curr;
	float point_type_next;
} vs_out;

void main()
{
	gl_Position = vec4(a_position_curr, 0.0, 1.0);
	vs_out.position_prev = vec4(a_position_prev, 0.0, 1.0);
	vs_out.position_next = vec4(a_position_next, 0.0, 1.0);
	vs_out.point_type_curr = a_point_type_curr;
	vs_out.point_type_next = a_point_type_next;
}
```

#### Geometry shader

```glsl
#version 330 core

layout (points) in;
layout (triangle_strip, max_vertices = 8) out;

uniform vec4 u_viewport;
uniform float u_pixel_width;
uniform float u_miter_limit;
uniform int u_cap_style; // flat, square, round
uniform int u_join_style; // bevel, miter, round

in VS_OUT {
	vec4 position_prev;
	vec4 position_next;
	float point_type_curr;
	float point_type_next;
} gs_in[];

noperspective out vec2 v_position;
noperspective out float v_point_type;
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

// We draw quad from current point to next one
void build_quad(vec4 prev, vec4 curr, vec4 next)
{
	vec2 screen_prev = project(prev);
	vec2 screen_curr = project(curr);
	vec2 screen_next = project(next);

	vec2 d1 = normalize(screen_curr - screen_prev);
	vec2 n1 = vec2(-d1.y, d1.x);
	vec2 d2 = screen_next - screen_curr;
	float segment_length = length(d2);
	d2 /= segment_length; // normalize
	vec2 n2 = vec2(-d2.y, d2.x);

	float w = u_pixel_width * 0.5;
	float point_type_curr = gs_in[0].point_type_curr;
	float point_type_next = gs_in[0].point_type_next;

	if (abs(point_type_curr) < 0.5) // current point is join, not start point
	{
		if (u_join_style == 0) // bevel join
		{
			// Bevel triangle
			float signed_z = w * sign(d1.x*d2.y - d2.x*d1.y);
			gl_Position = unproject(screen_curr - n1 * signed_z, curr.z, curr.w);
			v_position = vec2(0.0);
			v_point_type = 0.0;
			v_length = segment_length;
			v_radius = w;
			EmitVertex();
			gl_Position = unproject(screen_curr - n2 * signed_z, curr.z, curr.w);
			v_position = vec2(0.0);
			v_point_type = 0.0;
			v_length = segment_length;
			v_radius = w;
			EmitVertex();
			gl_Position = curr;
			v_position = vec2(0.0);
			v_point_type = 0.0;
			v_length = segment_length;
			v_radius = w;
			EmitVertex();
			EndPrimitive();
		}
		else if (u_join_style == 1) // miter join
		{
			// Miter quad
			float cos_a = dot(-d1, d2);
			float miter_limit = u_miter_limit * w;
			float miter_distance = w * sqrt((1.0 + cos_a) / (1.0 - cos_a));
			if (cos_a < 0.98 && miter_distance < miter_limit)
			{
				float signed_z = w * sign(d1.x*d2.y - d2.x*d1.y);
				vec2 first_point  = screen_curr - n1 * signed_z;
				vec2 second_point = screen_curr - n2 * signed_z;
				vec2 miter_point  = first_point + d1 * miter_distance;
				gl_Position = unproject(first_point, curr.z, curr.w);
				v_position = vec2(0.0);
				v_point_type = 0.0;
				v_length = segment_length;
				v_radius = w;
				EmitVertex();
				gl_Position = curr;
				v_position = vec2(0.0);
				v_point_type = 0.0;
				v_length = segment_length;
				v_radius = w;
				EmitVertex();
				gl_Position = unproject(miter_point, curr.z, curr.w);
				v_position = vec2(0.0);
				v_point_type = 0.0;
				v_length = segment_length;
				v_radius = w;
				EmitVertex();
				gl_Position = unproject(second_point, curr.z, curr.w);
				v_position = vec2(0.0);
				v_point_type = 0.0;
				v_length = segment_length;
				v_radius = w;
				EmitVertex();
				EndPrimitive();
			}
		}
		else // round join
		{
			vec2 p1 = screen_curr + n2 * w;
			vec2 p2 = p1 - d2 * w;
			vec2 p3 = screen_curr - n2 * w;
			vec2 p4 = p3 - d2 * w;

			gl_Position = unproject(p1, curr.z, curr.w);
			v_position = vec2(0.0, w);
			v_point_type = point_type_curr;
			v_length = segment_length;
			v_radius = w;
			EmitVertex();
			gl_Position = unproject(p2, curr.z, curr.w);
			v_position = vec2(-w, w);
			v_point_type = point_type_curr;
			v_length = segment_length;
			v_radius = w;
			EmitVertex();
			gl_Position = unproject(p3, curr.z, curr.w);
			v_position = vec2(0.0, -w);
			v_point_type = point_type_curr;
			v_length = segment_length;
			v_radius = w;
			EmitVertex();
			gl_Position = unproject(p4, curr.z, curr.w);
			v_position = vec2(-w, -w);
			v_point_type = point_type_curr;
			v_length = segment_length;
			v_radius = w;
			EmitVertex();
			EndPrimitive();
		}
	}

	vec2 curr_offset_x = vec2(0.0);
	vec2 next_offset_x = vec2(0.0);
	vec2 offset_y = n2 * w;
	float cap_offset_curr = 0.0;
	float cap_offset_next = 0.0;
	if (u_cap_style != 0) // not flat cap (square cap or round cap)
	{
		cap_offset_curr = w * point_type_curr;
		cap_offset_next = w * point_type_next;
		curr_offset_x = d2 * cap_offset_curr;
		next_offset_x = d2 * cap_offset_next;
	}

	// Quad
	gl_Position = unproject(screen_curr + offset_y + curr_offset_x, curr.z, curr.w); // left top
	v_position = vec2(cap_offset_curr, w);
	v_point_type = point_type_curr;
	v_length = segment_length;
	v_radius = w;
	EmitVertex();
	gl_Position = unproject(screen_curr - offset_y + curr_offset_x, curr.z, curr.w); // left bottom
	v_position = vec2(cap_offset_curr, -w);
	v_point_type = point_type_curr;
	v_length = segment_length;
	v_radius = w;
	EmitVertex();
	gl_Position = unproject(screen_next + offset_y + next_offset_x, next.z, next.w); // right top
	v_position = vec2(segment_length + cap_offset_next, w);
	v_point_type = point_type_next;
	v_length = segment_length;
	v_radius = w;
	EmitVertex();   
	gl_Position = unproject(screen_next - offset_y + next_offset_x, next.z, next.w); // right bottom
	v_position = vec2(segment_length + cap_offset_next, -w);
	v_point_type = point_type_next;
	v_length = segment_length;
	v_radius = w;
	EmitVertex();
	EndPrimitive();
}

void main()
{    
	build_quad(gs_in[0].position_prev, gl_in[0].gl_Position, gs_in[0].position_next);
}
```

#### Fragment shader

```glsl
#version 330 core

uniform int u_cap_style; // flat, square, round
uniform int u_join_style; // bevel, miter, round

out vec4 color;

noperspective in vec2 v_position;
noperspective in float v_point_type;
flat in float v_length;
flat in float v_radius;

void main()
{
	vec2 position = v_position;
	float point_type = v_point_type;
	float len = v_length;
	float radius = v_radius;

	bool is_join = abs(point_type) < 0.5;
	bool need_discard = !is_join && u_cap_style == 2 || is_join && u_join_style == 2;

	if (need_discard && position.x < 0.0)
	{
		float dist = length(position); // = distance(position, vec2(0.0))
		if (dist > radius)
			discard;
	}
	else if (need_discard && position.x > len)
	{
		float dist = length(vec2(position.x - len, position.y)); // = distance(position, vec2(len, 0.0))
		if (dist > radius)
			discard;
	}
	color = vec4(1.0, 0.0, 0.0, 0.0);
}
```

## Conclusion

<img src="{{ '/assets/img/polyline-joins.png' | relative_url }}">

We achieved polyline rendering with different joins.
In the next post we will cover polyline rendering with dash pattern.