---
layout: post
title: "Logotype SVG creation"
author: "Shtille"
categories: journal
tags: [2D,math,SVG]
---

In 2018 I've already described a [logotype model creation]({% post_url 2018-01-01-logotype-model-creation %}).
Now I have a need to make a SVG image logotype for site icon.

<img src="{{ '/assets/img/logo-2.png' | relative_url }}">

## Basic parameters

Torus is described by two radiuses _R_ and _r_, where:

$$ R = 10*r $$

Center of the first torus:

$$ C_1 = (0, 0) $$

Center of the second torus:

$$ C_2 = (-R\frac{\sqrt{3}}{2}, -R\frac{1}{2}) $$

Hemitoruses are enclosed with hemispheres with radiuses _r_.
The "eye" torus has length:

$$ L = R\frac{4}{5} $$

## SVG format details

Mozilla has an excellent [SVG format description](https://developer.mozilla.org/en-US/docs/Web/SVG/) on official MDN site.
We will use only following commands:
- MoveTo: M
- Elliptical Arc Curve: a

See [d attribute description](https://developer.mozilla.org/en-US/docs/Web/SVG/Attribute/d) for command details.

## Full logotype definition

$$ M x_0,y_0+(R-r) $$

$$ a (R-r),(R-r),0,0,1,0,(2r-2R) $$

$$ r,r,0,0,0,0,-2r $$

$$ (R+r),(R+r),0,0,0,0,2(R+r) $$

$$ M (x_0-R\frac{\sqrt{3}}{2}),(y_0-R\frac{1}{2}+r) $$

$$ a (R-r),(R-r),0,0,1,0,(2R-2r) $$

$$ r,r,0,0,0,0,2r $$

$$ (R+r),(R+r),0,0,0,0,-2(R+r) $$

$$ M (x_0-r\frac{\sqrt{3}}{2}),(y_0+r\frac{1}{2}) $$

$$ a \frac{R}{2},\frac{R}{2},0,0,0,-R\frac{4}{5}\frac{\sqrt{3}}{2},R\frac{4}{5}\frac{1}{2} $$

$$ \frac{R}{2},\frac{R}{2},0,0,0,R\frac{4}{5}\frac{\sqrt{3}}{2},-R\frac{4}{5}\frac{1}{2} $$

where $$ (x_0,y_0) $$ is the start position of our equations (first torus' center).

The logotype center position will be:

$$ x_c = x_0 - r\frac{\sqrt{3}}{2} - R\frac{2}{5}\frac{\sqrt{3}}{2} $$

$$ y_c = y_0 + r\frac{1}{2} + R\frac{2}{5}\frac{1}{2} $$

Let's determine maximum distance from center to edge _d_:

$$ d = y_c - y_{min} = r\frac{3}{2} + R\frac{6}{5} $$

And viewport will be: $$ (x_c-d, y_c-d, 2d, 2d) $$

## Final SVG code

```html
<svg viewBox="10.67 17.5 30 30" xmlns="http://www.w3.org/2000/svg">
  <!-- x0=30, y0=30 -->
  <!-- xc=25.67, yc=32.5 -->
  <!-- d=15 -->
  <path fill="currentColor"
    d="M 30 39
       a 9 9 0 0 1 0 -18
         1 1 0 0 0 0 -2
         11 11 0 0 0 0 22
       M 21.34 26
       a 9 9 0 0 1 0 18
         1 1 0 0 0 0 2
         11 11 0 0 0 0 -22
       M 29.134 30.5
       a 5 5 0 0 0 -6.9282 4
         5 5 0 0 0 6.9282 -4" />
</svg>
```