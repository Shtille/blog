---
layout: post
title: "First look at CMake"
author: "Shtille"
categories: journal
tags: [thoughts,CMake]
#image: cards.jpg
---

### Quick overview
CMake is now a standard layer for C/C++ development since people have become too lazy to write proper makefiles. CMake generates makefiles based on its own CMakeLists.txt files. Then you have to call make on your own to build your project.

### Pros and cons
#### Pros:
1. Easy to learn.
2. Easy to write CMakeLists.
3. Tunable.

#### Cons:
1. Makefile based build system links all libraries despite their up-to-date status.
2. CMake has some bugs on the old versions (I met it on versions less than 3.19).

The first drawback is avoidable by using [Ninja](https://ninja-build.org/). I strongly recommend to use it. It does well with concurrent build.

The second drawback is avoidable by using the latest version. If you use Android Studio don't hesitate to use the latest available CMake in SDK Manager.
By default Android Studio installs CMake version 3.10 that has some bugs: it cant mix *target_link_libraries* with *set_target_properties* with link flags. It might have other bugs but this is the one I met.
