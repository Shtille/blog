---
layout: post
title: "libjpeg vs libpng performance comparison"
author: "Shtille"
categories: journal
tags: [thoughts,C++]
---

Done buffer save/load tests for these libraries, and results are following: 

- JPEG test took 0.017918 seconds
- PNG test took 0.088861 seconds

So _libjpeg_ is 5.5 times faster than _libpng_.