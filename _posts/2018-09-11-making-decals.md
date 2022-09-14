---
layout: post
title: "Making Decals (OpenGL Programming)"
author: "Shtille"
categories: journal
tags: [thoughts,OpenGL]
---

Suppose you’re drawing a complex three-dimensional picture using depth-buffering to eliminate the hidden surfaces. Suppose further that one part of your picture is composed of coplanar figures A and B, where figure B is a sort of decal that should always appear on top of figure A.
Your first approach might be to draw figure B after you’ve drawn figure A, setting the depth-buffering function to replace fragments when their depth is greater than or equal to the value stored in the depth buffer (GL_GEQUAL). Owing to the finite precision of the floating-point representations of the vertices, however, round-off error can cause figure B to be sometimes a bit in front of and sometimes a bit behind figure A. Here’s one solution to this problem:
1.    Disable the depth buffer for writing, and render A.
2.    Enable the depth buffer for writing, and render B.
3.    Disable the color buffer for writing, and render A again.
4.    Enable the color buffer for writing.
Note that during the entire process, the depth-buffer test is enabled. In Step 1, A is rendered wherever it should be, but none of the depth-buffer values are changed; thus, in Step 2, wherever B appears over A, B is guaranteed to be drawn. Step 3 simply makes sure that all of the depth values under A are updated correctly, but since RGBA writes are disabled, the color pixels are unaffected. Finally, Step 4 returns the system to the default state (writing is enabled in both the depth buffer and the color buffer).

If a stencil buffer is available, the following, simpler technique can be used:
1.    Configure the stencil buffer to write 1 if the depth test passes, and 0 otherwise. Render A.
2.    Configure the stencil buffer to make no stencil value change, but to render only where stencil values are 1. Disable the depth-buffer test and its update. Render B.
With this method, it’s not necessary to initialize the contents of the stencil buffer at any time, because the stencil values of all pixels of interest (that is, those rendered by A) are set when A is rendered. Be sure to reenable the depth test and disable the stencil test before additional polygons are drawn.

Original text has been provided by [what-when-how.com](http://what-when-how.com/opengl-programming-guide/making-decals-opengl-programming/)