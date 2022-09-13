---
layout: post
title: "Program to test intersections"
author: "Shtille"
categories: journal
tags: [C++,samples]
#image: cards.jpg
---

Made a program to test intersections:

```cpp
#include <iostream>
#include <string>
#include <cmath>

struct vec3 {
    vec3(float x) : x(x), y(x), z(x) {}
    vec3(float x, float y, float z) : x(x), y(y), z(z) {}
    vec3 operator +(const vec3& v) const { return vec3(x + v.x, y + v.y, z + v.z); }
    vec3 operator -(const vec3& v) const { return vec3(x - v.x, y - v.y, z - v.z); }
    vec3 operator *(const vec3& v) const { return vec3(x * v.x, y * v.y, z * v.z); }
    vec3 operator *(const float v) const { return vec3(x * v, y * v, z * v); }
    vec3 operator /(const vec3& v) const { return vec3(x / v.x, y / v.y, z / v.z); }
    float x,y,z;
};
struct Ray {
    vec3 origin;
    vec3 direction;
};
struct Sphere {
	vec3 position;
	float radius;
};
struct Box {
	vec3 min;
	vec3 max;
};

float dot(const vec3& v1, const vec3& v2)
{
    return v1.x * v2.x + v1.y * v2.y + v1.z * v2.z;
}
vec3 min(const vec3& v1, const vec3& v2)
{
    return vec3(
        (v1.x < v2.x) ? v1.x : v2.x,
        (v1.y < v2.y) ? v1.y : v2.y,
        (v1.z < v2.z) ? v1.z : v2.z
        );
}
vec3 max(const vec3& v1, const vec3& v2)
{
    return vec3(
        (v1.x > v2.x) ? v1.x : v2.x,
        (v1.y > v2.y) ? v1.y : v2.y,
        (v1.z > v2.z) ? v1.z : v2.z
        );
}
float min(float x1, float x2)
{
    return (x1 < x2) ? x1 : x2;
}
float max(float x1, float x2)
{
    return (x1 > x2) ? x1 : x2;
}
bool IntersectSphere(const Sphere& sphere, const Ray& ray, float& t)
{
    std::cout << "sphere" << "\n";
	vec3 op = sphere.position - ray.origin;
    float b = dot(op, ray.direction);
    float det = b * b - dot(op, op) + sphere.radius * sphere.radius;
    if (det < 0.0f) return false;

    det = sqrt(det);
    t = b - det;
    if (t < 0.0f) t = b + det;
    if (t < 0.0f) return false;
    return true;
}
bool IntersectBox(const Box& box, const Ray& ray, float& t)
{
    std::cout << "box" << "\n";
	vec3 tMin = (box.min - ray.origin) / ray.direction;
	vec3 tMax = (box.max - ray.origin) / ray.direction;
	vec3 t1 = min(tMin, tMax);
	vec3 t2 = max(tMin, tMax);
	float tNear = max(max(t1.x, t1.y), t1.z);
	float tFar = min(min(t2.x, t2.y), t2.z);
	std::cout << "tNear = " << tNear << "\n";
	std::cout << "tFar = " << tFar << "\n";
	if (tNear < tFar)
	{
	    t = (tNear > 0.0f) ? tNear : tFar;
	    return true;
	}
	else
	    return false;
}

int main()
{
    Sphere sphere = {vec3(0.0f), 1.0f};
    Box box = {vec3(-1.0f), vec3(1.0f)};
    Ray ray = {vec3(0.0f, 0.5f, 0.0f), vec3(1.0f, 0.0f, 0.0f)};
    float t = -1000.0f;
    //bool hit = IntersectBox(box, ray, t);
    bool hit = IntersectSphere(sphere, ray, t);
    std::cout << "hit = " << hit << "\n";
    std::cout << "t = " << t << "\n";
}
```