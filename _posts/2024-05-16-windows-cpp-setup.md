---
layout: post
title: "Setup for Windows C++ development"
author: "Shtille"
categories: journal
tags: [code,bash]
---

## Basic tools

To start C++ developing for Windows I'd recommend using [MSYS](https://www.msys2.org/) toolchain.
1. Install it via installer.
2. Run "MSYS2 MSYS" console and get needed packages for Mingw-w64:
```bash
pacman -S --needed base-devel mingw-w64-ucrt-x86_64-toolchain
```
3. Add "%MSYS_PATH%\ucrt64\bin" to *PATH* variable.

### Check toolchain installation
```bash
gcc --version
g++ --version
gdb --version
```

## *ffmpeg* installation

1. Get sources
```bash
git clone https://github.com/FFmpeg/FFmpeg
```
2. Run "MSYS MINGW64" console.
3. Configure and build:
```bash
./configure --disable-ffplay --disable-ffprobe --disable-ffmpeg --enable-static 
make
make install
```