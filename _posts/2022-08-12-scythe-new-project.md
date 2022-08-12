---
layout: post
title: "Make new project with scythe"
author: "Shtille"
categories: journal
tags: [C++,samples]
#image: cards.jpg
---

The following guide will describe how to make a new project based on scythe™ framework.

---

1. Initialize the project with git.
```bash
mkdir <project_name>
cd <project_name>
git init
```
2. Add dependencies.
```bash
mkdir deps
git submodule add https://github.com/Shtille/scythe-thirdparty.git deps/thirdparty
git submodule add https://github.com/Shtille/scythe.git deps/scythe
```
3. Add sources.
```bash
mkdir src
cp deps/scythe/src/example/desktop.cpp src/
```
4. Add dependent resources into _data_ directory.
```bash
rsync -avz deps/scythe/data/ ./data
```
5. Create CMakeLists.txt build file.
```cmake
cmake_minimum_required(VERSION 3.13 FATAL_ERROR)

# Some settings
set(BINARY_PATH "${CMAKE_CURRENT_SOURCE_DIR}/bin")
set(SCYTHE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/deps/scythe")
set(SCYTHE_THIRDPARTY_DIR "${CMAKE_CURRENT_SOURCE_DIR}/deps/thirdparty")

# Dependencies
add_subdirectory(deps/thirdparty)
add_subdirectory(deps/scythe)

# The project itself
project(Demo)

set(CMAKE_CXX_STANDARD 14)
set(SRC_DIRS
	src
)
set(include_directories
	${SCYTHE_PATH}/include
	${SCYTHE_PATH}/src
)
#set(defines )
set(libraries
	scythe
)

foreach(DIR ${SRC_DIRS})
	file(GLOB DIR_SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/${DIR}/*.cpp)
	set(SRC_FILES ${SRC_FILES} ${DIR_SOURCE})
endforeach(DIR)

if (WIN32)
	add_executable(${PROJECT_NAME} WIN32 ${SRC_FILES})
else()
	add_executable(${PROJECT_NAME} ${SRC_FILES})
endif()
target_include_directories(${PROJECT_NAME} PRIVATE ${include_directories})
#target_compile_definitions(${PROJECT_NAME} PRIVATE ${defines})
target_link_libraries(${PROJECT_NAME} PRIVATE ${libraries})

install(TARGETS ${PROJECT_NAME}
		RUNTIME DESTINATION ${BINARY_PATH})
```
6. Build and install project with CMake.
```bash
mkdir build
cd build
cmake ..
cd ..
cmake --build build
cmake --install build
```
7. Run your application.
```bash
./bin/Demo
```