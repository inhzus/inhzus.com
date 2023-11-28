---
title: "C++ Build / CMake Tricks"
date: 2023-09-19T18:16:45+08:00
tags: [C++]
description: Tricks around C++ building and CMake
---

## Multiple Static -> Static

```
MSVC: lib.exe
Linux: ar -rcT libgaea.a libgaea_base.a libgaea_config.a
Apple:
  lipo -create libgaea_base.a libgaea_config.a -output libgaea.a
  libtool

```

XMake supports merging static libraries: [Ref](https://xmake.io/#/guide/project_examples?id=merge-static-libraries)

## Multiple Static -> Shared

<del>There was a CMake snippet which is a cross-platform solution on gist. Unfortunately cannot be found again.<del/>

Now CMake provides a generator expression, [LINK_LIBRARY](https://cmake.org/cmake/help/v3.24/manual/cmake-generator-expressions.7.html#genex:LINK_LIBRARY), to solve this issue. 🎉

```
MSVC: /WHOLE_ARCHIVE
gcc: --whole-archive
Clang: --force_load

```

## Circular dependencies

```
CMake: target_link_libraries(foobar PRIVATE $<LINK_GROUP:RESCAN,foo,bar>)
# https://cmake.org/cmake/help/latest/variable/CMAKE_LINK_GROUP_USING_FEATURE.html#predefined-features
```

<br/>

## Other Problems

1. CMake will treat all $PATH and their parent paths as system path (e.g. /opt/brew/bin & /opt/brew).
   This can be problematic while i only use those binary executables in paths, and do not want to use libraries there.
   
   Well we can use [CMAKE_IGNORE_PREFIX_PATH](https://cmake.org/cmake/help/latest/variable/CMAKE_IGNORE_PREFIX_PATH.html#variable:CMAKE_IGNORE_PREFIX_PATH)(CMake 3.23+) or [CMAKE_IGNORE_PATH](https://cmake.org/cmake/help/latest/variable/CMAKE_IGNORE_PATH.html#variable:CMAKE_IGNORE_PATH) to ignore them, or else we may need to make $PATH clean before run cmake configuring.
