---
layout: post
title: "Polyline performance"
author: "Shtille"
categories: journal
tags: [C++,GLSL,samples]
---

## Overview

I decided to check if geometry shader provides better performance for polyline rendering comparing to normal one.
For test application I gonna render 3 segments as quads. The code is about to be pretty simple.

## Without geometry shader

Traditional way (only vertex and fragment shaders) has been implemented previously. Let's recall it.

### Application code

#### Data creation

```cpp
bool OldQuadDrawer::CreateData(const PointArray& points)
{
	uint32_t num_points = static_cast<uint32_t>(points.size());
	if (num_points < 2) return false;
	uint32_t num_segments = num_points - 1;
	uint32_t total_segments = num_segments + 2;

	num_vertices_ = 4 * total_segments;
	vertices_array_ = new uint8_t[num_vertices_ * sizeof(Vertex)];
	Vertex* vertices = reinterpret_cast<Vertex*>(vertices_array_);

	num_indices_ = 6 * num_segments;
	index_size_ = sizeof(uint32_t);
	indices_array_ = new uint8_t[num_indices_ * index_size_];
	uint32_t* indices = reinterpret_cast<uint32_t*>(indices_array_);

	// Positions
	uint32_t n = 0;
	Point first_point, last_point;
	first_point[0] = points[0][0] + (points[0][0] - points[1][0]);
	first_point[1] = points[0][1] + (points[0][1] - points[1][1]);
	last_point[0] = points[num_points - 1][0] + (points[num_points - 1][0] - points[num_points - 2][0]);
	last_point[1] = points[num_points - 1][1] + (points[num_points - 1][1] - points[num_points - 2][1]);
	vertices[n++].position = first_point;
	vertices[n++].position = first_point;
	vertices[n++].position = points[0];
	vertices[n++].position = points[0];
	for (uint32_t i = 0; i < num_segments; ++i)
	{
		vertices[n++].position = points[i];
		vertices[n++].position = points[i];
		vertices[n++].position = points[i+1];
		vertices[n++].position = points[i+1];
	}
	vertices[n++].position = points[num_points - 1];
	vertices[n++].position = points[num_points - 1];
	vertices[n++].position = last_point;
	vertices[n++].position = last_point;

	// Texcoords
	n = 0;
	for (uint32_t i = 0; i < total_segments; ++i)
	{
		vertices[n++].texcoord = { 0.0f,  1.0f};
		vertices[n++].texcoord = { 0.0f, -1.0f};
		vertices[n++].texcoord = { 0.0f,  1.0f};
		vertices[n++].texcoord = { 0.0f, -1.0f};
	}

	// Indices
	n = 0;
	for (uint32_t i = 0; i < num_segments; ++i)
	{
		// 0-1-2
		indices[n++] = 4*i+0;
		indices[n++] = 4*i+1;
		indices[n++] = 4*i+2;
		// 2-1-3
		indices[n++] = 4*i+2;
		indices[n++] = 4*i+1;
		indices[n++] = 4*i+3;
	}

	return true;
}
```

#### Attributes layout

```cpp
	const GLsizei stride = sizeof(Vertex);
	const uint8_t* base = nullptr;
	const uint8_t* curr_offset = base + 4*stride; // offset to start of actual current data
	const uint8_t* prev_offset = curr_offset - 2*stride;
	const uint8_t* next_offset = curr_offset + 2*stride;
	const uint8_t* texcoord_offset = curr_offset + sizeof(Point);
	glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, stride, prev_offset); // vec2 a_prev
	glEnableVertexAttribArray(0);
	glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, stride, curr_offset); // vec2 a_curr
	glEnableVertexAttribArray(1);
	glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, stride, next_offset); // vec2 a_next
	glEnableVertexAttribArray(2);
	glVertexAttribPointer(3, 2, GL_FLOAT, GL_FALSE, stride, texcoord_offset); // vec2 a_texcoord;
	glEnableVertexAttribArray(3);
```

#### Rendering

```cpp
void OldQuadDrawer::Render()
{
	ActivateShader();

	glBindVertexArray(vertex_array_object_);
	glDrawElements(GL_TRIANGLES, num_indices_, GL_UNSIGNED_INT, nullptr);
	glBindVertexArray(0);

	DeactivateShader();
}
```

### Shader code

#### Vertex shader

```glsl
#version 330 core

layout(location = 0) in vec2 a_prev;
layout(location = 1) in vec2 a_curr;
layout(location = 2) in vec2 a_next;
layout(location = 3) in vec2 a_texcoord;

uniform vec4 u_viewport;
uniform float u_width;

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
	vec4 clip_curr = vec4(a_curr, 0.0, 1.0);
	vec4 clip_prev = vec4(a_prev, 0.0, 1.0);
	vec4 clip_next = vec4(a_next, 0.0, 1.0);

	vec2 curr = project(clip_curr);
	vec2 prev = project(clip_prev);
	vec2 next = project(clip_next);

	vec2 direction = normalize(next - prev);
	vec2 normal = vec2(-direction.y, direction.x);

	float w = u_width * 0.5;
	vec2 offset_x = direction * (a_texcoord.x * w);
	vec2 offset_y = normal    * (a_texcoord.y * w);
	vec2 screen = curr + offset_x + offset_y;

	gl_Position = unproject(screen, clip_curr.z, clip_curr.w);
}
```

#### Fragment shader

```glsl
#version 330 core

out vec4 color;

void main()
{
	color = vec4(1.0, 0.0, 0.0, 0.0);
}
```

## With geometry shader

Geometry shader is more optimal memory-wise.

### Application code

#### Data creation

```cpp
bool QuadDrawer::CreateData(const PointArray& points)
{
	uint32_t num_points = static_cast<uint32_t>(points.size());
	if (num_points < 2) return false;

	num_vertices_ = num_points;
	vertices_array_ = new uint8_t[num_vertices_ * sizeof(Vertex)];
	Vertex* vertices = reinterpret_cast<Vertex*>(vertices_array_);

	for (uint32_t i = 0; i < num_vertices_; ++i)
	{
		Vertex& vertex = vertices[i];
		const Point& point = points[i];

		vertex.position = {point[0], point[1]};
	}

	return true;
}
```

#### Attributes layout

```cpp
	const GLsizei stride = sizeof(Vertex);
	const uint8_t* base = nullptr;
	const uint8_t* curr_offset = base;
	const uint8_t* next_offset = curr_offset + stride;
	glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, stride, curr_offset); // vec2 a_position_curr
	glEnableVertexAttribArray(0);
	glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, stride, next_offset); // vec2 a_position_next
	glEnableVertexAttribArray(1);
```

#### Rendering

```cpp
void QuadDrawer::Render()
{
	ActivateShader();

	glBindVertexArray(vertex_array_object_);
	glDrawArrays(GL_POINTS, 0, num_vertices_-1);
	glBindVertexArray(0);

	DeactivateShader();
}
```

### Shader code

#### Vertex shader

```glsl
#version 330 core

layout (location = 0) in vec2 a_position_curr;
layout (location = 1) in vec2 a_position_next;

out VS_OUT {
	vec4 position_next;
} vs_out;

void main()
{
	gl_Position = vec4(a_position_curr, 0.0, 1.0);
	vs_out.position_next = vec4(a_position_next, 0.0, 1.0);
}
```

#### Geometry shader

```glsl
#version 330 core

layout (points) in;
layout (triangle_strip, max_vertices = 4) out;

uniform vec4 u_viewport;
uniform float u_pixel_width;

in VS_OUT {
	vec4 position_next;
} gs_in[];

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

void build_quad(vec4 current, vec4 next)
{
	vec2 screen_current = project(current);
	vec2 screen_next = project(next);

	vec2 direction = normalize(screen_next - screen_current);
	vec2 normal = vec2(-direction.y, direction.x);

	float w = u_pixel_width * 0.5;

	gl_Position = unproject(screen_current + normal * w, current.z, current.w); // left top
	EmitVertex();   
	gl_Position = unproject(screen_current - normal * w, current.z, current.w); // left bottom
	EmitVertex();
	gl_Position = unproject(screen_next + normal * w, next.z, next.w); // right top
	EmitVertex();   
	gl_Position = unproject(screen_next - normal * w, next.z, next.w); // right bottom
	EmitVertex();
	EndPrimitive();
}

void main()
{    
	build_quad(gl_in[0].gl_Position, gs_in[0].position_next);
}
```

#### Fragment shader

```glsl
#version 330 core

out vec4 color;

void main()
{
	color = vec4(1.0, 0.0, 0.0, 0.0);
}
```

## Performance capture

For performance capture we use `glBeginQuery` - `glEndQuery` block:

```cpp
void TestIteration()
{
	poly::TimeElapsedQuery query;

	std::cout << "--- iteration begin" << std::endl;
	{
		query.Begin();
		old_drawer->Render();
		query.End();
		if (query.GetResult(true))
		{
			std::cout << "old took " << query.GetElapsedTime() << " ns" << std::endl;
		}
	}
	{
		query.Begin();
		new_drawer->Render();
		query.End();
		if (query.GetResult(true))
		{
			std::cout << "new took " << query.GetElapsedTime() << " ns" << std::endl;
		}
	}
	std::cout << "--- iteration end" << std::endl;
}
```

## Results

For three iterations we get following results:

```
--- iteration begin
old took 21268 ns
new took 13936 ns
--- iteration end
--- iteration begin
old took 19760 ns
new took 13312 ns
--- iteration end
--- iteration begin
old took 18616 ns
new took 13312 ns
--- iteration end
```

So quad drawer with geometry shader is about 1.5 times faster than a version without geometry shader.

### Conclusion

We can conclude that geometry shader usage provides better performance for polyline rendering.