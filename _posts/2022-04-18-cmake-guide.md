---
layout: post
title: "CMake quick guide"
author: "Shtille"
#categories: journal
tags: [thoughts,CMake]
#image: cards.jpg
---

### Quick overview
CMake commands are not case sensitive, so you may use any version you like the most.
Translating commands to another line doesn't require additional characters like '\' in Makefile.
Commentaries are made with '#' starting symbol (like in Makefile).

### Variables
Variables are being set with following command:
```cmake
set(value 10)
```
To clear the value one should use:
```cmake
unset(value)
```
To access variable use its name in parentheses with dollar prefix:
```cmake
${value}
```
Variables only visible on the scope they're being set and below.
The variable with the same name on any scope below will hide upper variable (like on any programmic language, C/C++/JavaScript as example).
If you need to make variable be visible in the upper scope use:
```cmake
set(value 10 PARENT_SCOPE)
```
#### Lists
Lists are made with space divided values:
```cmake
set(my_sources first.cpp second.cpp third.cpp)
```
To append value to list there are two options:
```cmake
list(APPEND my_sources fourth.cpp)
```
or
```cmake
set(my_sources ${my_sources} fourth.cpp)
```
#### Prebuilt variables
List of useful prebuilt variables:
- *CMAKE_CURRENT_SOURCE_DIR*
- *CMAKE_CURRENT_BINARY_DIR*
#### Setting C++ standard version
```cmake
set(CMAKE_CXX_STANDARD 11)
```

### Project structure
You need to create CMakeLists.txt file in each directory you have target and intermediate files for easy build, including root file.
Subdirectories can be included via command *add_subdirectory*.

So the root file should have following look:
```cmake
cmake_minimum_required(VERSION 3.19)

project(my_project)

add_subdirectory(my_static_library_dir)
add_subdirectory(my_shared_library_dir)
add_subdirectory(my_executable_dir)
add_subdirectory(thirdparty_dir)
```
Subdirectories can have many layers:
```cmake
project(thirdparty)

add_subdirectory(zlib)
add_subdirectory(jpeg)
add_subdirectory(png)
```

### Types of targets
- Executable
- Static library
- Shared library
- Object library
- Custom target

#### Executable
```cmake
project(my_project)
...
add_executable(${PROJECT_NAME} ${source_files})
```
#### Static library
```cmake
project(my_lib)
...
add_library(${PROJECT_NAME} STATIC ${source_files})
```
#### Shared library
Shared library requires to link dependencies.
```cmake
project(my_lib)
...
add_library(${PROJECT_NAME} SHARED ${source_files})
target_link_libraries(${PROJECT_NAME} PUBLIC ${libraries})
```
#### Object library
Object library is used only to compile object files and not generate static/dynamic libraries output.
```cmake
project(my_lib)
...
add_library(${PROJECT_NAME} OBJECT ${source_files})
```
#### Custom target
Example of usage of custom target:
```cmake
set(OPENGL_OLD_SHADERS
	${CMAKE_CURRENT_SOURCE_DIR}/AppUtils/RenderingContexts/OpenGL2/OpenGLContextShader_fsh.cpp
	${CMAKE_CURRENT_SOURCE_DIR}/AppUtils/RenderingContexts/OpenGL2/OpenGLContextShader_vsh.cpp
)

if (WIN32)
	message(FATAL_ERROR "Provide target for building shaders for Windows")
else()
	add_custom_command(
		OUTPUT ${OPENGL_OLD_SHADERS}
		COMMAND ${CMAKE_COMMAND} -E env ${EML_ROOT}/build/buildShaders.sh ${EML_ROOT}/AppUtils/RenderingContexts/OpenGL2/ ${EML_ROOT}/AppUtils/RenderingContexts/OpenGL2/ OpenGLContextShader.fsh
		COMMAND ${CMAKE_COMMAND} -E env ${EML_ROOT}/build/buildShaders.sh ${EML_ROOT}/AppUtils/RenderingContexts/OpenGL2/ ${EML_ROOT}/AppUtils/RenderingContexts/OpenGL2/ OpenGLContextShader.vsh
		DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/AppUtils/RenderingContexts/OpenGL2/OpenGLContextShader.fsh
		        ${CMAKE_CURRENT_SOURCE_DIR}/AppUtils/RenderingContexts/OpenGL2/OpenGLContextShader.vsh
		COMMENT "Building shaders"
		VERBATIM
	)
endif()

add_custom_target(build_shaders DEPENDS ${OPENGL_OLD_SHADERS})
```

### Linking libraries
So as you've seen before to link libraries simply use:
```cmake
project(my_lib)
...
target_link_libraries(${PROJECT_NAME} PUBLIC ${libraries})
```
after target (executable, library) was specified.
In new versions of CMake you may not set specific order of libraries to correct the linkage, also CMake automatically collects all the required libraries between subprojects.
The more information about link function read at [its manual](https://cmake.org/cmake/help/latest/command/target_link_libraries.html).
#### Link to Mac OS X / iOS frameworks
```cmake
project(my_lib)
...
target_link_libraries(${PROJECT_NAME} PUBLIC "-framework Cocoa")
```

### Include directories
Similar way handled include directories:
```cmake
project(my_lib)
...
target_include_directories(${PROJECT_NAME} PRIVATE ${directories})
```
Read [the manual](https://cmake.org/cmake/help/latest/command/target_include_directories.html) for more information.

### Compile definitions
```cmake
project(my_lib)
...
target_compile_definitions(${PROJECT_NAME} PRIVATE ${defines})
```
Read [the manual](https://cmake.org/cmake/help/latest/command/target_compile_definitions.html) for more information.

### Extended programming
#### Message logging
Use *message* command to log information:
```cmake
message(STATUS "My defines: " ${defines})
```
Read [the manual](https://cmake.org/cmake/help/latest/command/message.html) for more information.
#### If case
Useful when choose the platform target:
```cmake
if (WIN32)
	message(STATUS "WIN32")
elseif(ANDROID)
	message(STATUS "ANDROID")
else()
	message(STATUS "OTHER")
endif()
```
#### For cycle
Following code parses directories for *.c* and *.cpp* files and fills *MODULE_SOURCES* variable:
```cmake
foreach(DIR ${MODULE_DIRS})
  file(GLOB DIR_SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/${DIR}/*.c ${CMAKE_CURRENT_SOURCE_DIR}/${DIR}/*.cpp)
  set(MODULE_SOURCES ${MODULE_SOURCES} ${DIR_SOURCE})
endforeach(DIR)
```

### Building the code
```bash
cd <project directory>
cmake -S . -B build
cmake --build build
```
