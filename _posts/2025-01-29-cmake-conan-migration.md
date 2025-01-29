---
layout: post
title: "CMake based project migration to Conan"
author: "Shtille"
categories: journal
tags: [thoughts,cmake,conan]
---

### Overview
[Conan](https://conan.io/) is pretty popular package manager. It's very useful to not keep all you thirdparty dependecies up-to-date manually.

#### Pros
* Very easy to update dependencies

#### Cons
* Another level of abstraction above CMake
* Additional learning curve to Conan's API
* Dependencies may pull own dependencies normally you would rid of
* Need to configure each dependency via conanfile

### Migration

#### Project build

Typical project build with CMake looks like:
```bash
mkdir build && cd build
cmake ..
cmake --build .
```

With Conan at first we need to make sure that our profile is configured:
```bash
conan profile detect --force
```

Then every build will use CMake profiles instead of typical builds:
```bash
conan install . --output-folder=build --build=missing
cmake --preset conan-default
cmake --build --preset conan-release
```

### Conclusion
Hence it's a popular solution for big companies to package libraries, ordinary user might find it pretty complicated.
Here I didn't even cover packaging part. I think I gonna update the post later...