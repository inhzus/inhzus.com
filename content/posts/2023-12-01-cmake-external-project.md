---
title: "CMake: How to build external projects and use as imported targets"
date: 2023-12-01T12:15:31+08:00
description: "CMake: How to build external projects and use as imported targets"
tags: [CMake]
---

## Intro

#### Q: `ExternalProject` or `add_subdirectory`/`FetchContent` when no download need?

we use `ExternalProject` here to get **maximum isolation** between the thirdparty library and our codebase, especially because CMake scripts of our base project is quiet messy and customized and may pollute the thirdparty building if using `add_subdirectory`.

There're two steps to do:

- Use `ExternalProject_Add` to configure, build and install the dependency.
- Setup the dependency as an **imported library** for other targets to "link".

Here I mainly record the pits that have been encountered.

## Code
```CMake
# CMakeLists.txt
include(brpc.cmake)

# brpc.cmake
set(EXTERN_BRPC_PREFIX ${PROJECT_BINARY_DIR}/extern-brpc)
set(BRPC_INSTALL_PREFIX ${EXTERN_BRPC_PREFIX})

set(BRPC_CMAKE_ARGS -DCMAKE_PREFIX_PATH=${CMAKE_PREFIX_PATH}
                    -DCMAKE_INSTALL_PREFIX=${BRPC_INSTALL_PREFIX})

# Inherits ccache used by the base project
get_property(BRPC_LAUNCH_COMPILE GLOBAL PROPERTY RULE_LAUNCH_COMPILE)
if(NOT BRPC_LAUNCH_COMPILE STREQUAL "")
  list(APPEND BRPC_CMAKE_ARGS
       -DCMAKE_CXX_COMPILER_LAUNCHER=${BRPC_LAUNCH_COMPILE})
endif()
get_property(BRPC_LAUNCH_LINK GLOBAL PROPERTY RULE_LAUNCH_LINK)
if(NOT BRPC_LAUNCH_LINK STREQUAL "")
  list(APPEND BRPC_CMAKE_ARGS -DCMAKE_CXX_LINKER_LAUNCHER=${BRPC_LAUNCH_LINK})
endif()

include(ExternalProject)
ExternalProject_Add(
  extern-brpc
  PREFIX ${EXTERN_BRPC_PREFIX}
  SOURCE_DIR ${PROJECT_SOURCE_DIR}/brpc
  CMAKE_ARGS ${BRPC_CMAKE_ARGS}
  BUILD_BYPRODUCTS ${BRPC_INSTALL_PREFIX}/lib64/libbrpc.a)

add_library(vespa-brpc STATIC IMPORTED GLOBAL)
add_dependencies(vespa-brpc extern-brpc)
set(BRPC_INCLUDE_DIRS ${BRPC_INSTALL_PREFIX}/include)
# This is a workaround for the fact that included directories of an imported
# target should exist in the filesystem already at the configuration time.
# ref: https://gitlab.kitware.com/cmake/cmake/-/issues/15052
file(MAKE_DIRECTORY ${BRPC_INCLUDE_DIRS})
set_target_properties(
  vespa-brpc
  PROPERTIES IMPORTED_LOCATION ${BRPC_INSTALL_PREFIX}/lib64/libbrpc.a
             INTERFACE_INCLUDE_DIRECTORIES ${BRPC_INCLUDE_DIRS}
             INTERFACE_SYSTEM_INCLUDE_DIRECTORIES ${BRPC_INCLUDE_DIRS})
target_link_libraries(vespa-brpc INTERFACE gflags leveldb)
```


## ExternalProject

#### Q: How to use directory locally instead of downloading one?

Use `ExternalProject_Add` like this, and that's all.

```CMake
ExternalProject_Add(
	foo
	SOURCE_DIR /path/to/foo
)
```



#### Q: How do the following procedure get the products of the external project?

Simply put: INSTALL_DIR, and how?

We've known that the `ExternalProject_Add` will do configure, build and install for the external source during the build stage of our base project. The key is installing to a specific path, and we get that path later. There are two methods:

1. Predefine the install prefix path

Let's say it's `${PROJECT_BINARY_DIR}/extern-brpc`. (Put it the the binary dir for cleaning, and avoid conflicting of the existing paths).

```CMake
set(BRPC_INSTALL_PREFIX ${PROJECT_BINARY_DIR}/extern-brpc)
ExternalProject_Add(
	extern-brpc
	SOURCE_DIR extern-brpc
	CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${BRPC_INSTALL_PREFIX}
	# Specify as an arg for CMake
)
# example usage:
# include_directories(${BRPC_INSTALL_PREFIX}/include)
```

2. Get the `INSTALL_DIR` afterwards

The command provides a builtin variable named `<INSTALL_DIR>`. And we can get it by `ExternalProject_Get_property`.

```CMake
ExternalProject_Add(
	extern-brpc
	SOURCE_DIR extern-brpc
	CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
	# The builtin variable <INSTALL_DIR>
)
ExternalProject_Get_Property(extern-brpc INSTALL_DIR)
# include_directories(${INSTALL_DIR}/include)
```

- Pros: This is a builtin feature which may make the script a little bit cleaner
- Cons: The property is set into `INSTALL_DIR` which name is so common that it may pollute outside



#### Q: Is `BUILD_BYPRODUCTS` a must?

Yes if u use `Ninja` generator, or else Ninja will have no idea where the library binary file comes from. and it will produce such error:

```
ninja: error: '/path/to/libbrpc.a', needed by 'Project',
       missing and no known rule to make it
```

Ref: [Stackoverflow: ExternalProject_Add: BUILD_BYPRODUCTS](https://stackoverflow.com/questions/40314785/linking-against-an-externalproject-add-dependency-in-cmake)

CMake doc also explains:

> This may also be required to explicitly declare dependencies when using the [`Ninja`](https://cmake.org/cmake/help/latest/generator/Ninja.html#generator:Ninja) generator.



## Add Imported Target

#### Q: Why to manually create the installed include directory?

This is a workaround for the fact that the `INTERFACE_INCLUDE_DIRECTORIES` must be existed when do configuring (while the `INCLUDE_DIRECTORIES` not). The issue has been unresolved for 7 years: https://gitlab.kitware.com/cmake/cmake/-/issues/15052. Fortunately it's simple to handle.

## Bonus

#### Q: `CMAKE_SOURCE_DIR` vs `PROJECT_SOURCE_DIR` vs `CMAKE_CURRENT_SOURCE_DIR`?

Basically: `CMAKE_SOURCE_DIR` >= `PROJECT_SOURCE_DIR` >= `CMAKE_CURRENT_SOURCE_DIR`

`CMAKE_SOURCE_DIR` indicates the root path where CMake runs at.

`PROJECT_SOURCE_DIR` is the first parent path which CMakeLists declares `project`, thus this is always the root path of a thirdparty library.

`CMAKE_CURRENT_SOURCE_DIR` is the location of the `CMakeLists.txt` who uses the variable.

The relationship among `CMAKE_BINARY_DIR`, `PROJECT_BINARY_DIR` and `CMAKE_CURRENT_BINARY_DIR` are similar.



#### Q: `INTERFACE_SYSTEM_INCLUDE_DIRECTORIES` & `INTERFACE_INCLUDE_DIRECTORIES`

As [CMake Doc](https://cmake.org/cmake/help/latest/prop_tgt/INTERFACE_SYSTEM_INCLUDE_DIRECTORIES.html) goes:

> Adding directories to this property **marks** directories as system directories which otherwise would be used in a non-system manner.

So to use system include, we need to set both `INTERFACE_INCLUDE_DIRECTORIES` and `INTERFACE_SYSTEM_INCLUDE_DIRECTORIES`.

## Reference

- [ExternalProject - CMake Documentation](https://cmake.org/cmake/help/latest/module/ExternalProject.html)
- [add_library - CMake Documentation](https://cmake.org/cmake/help/latest/command/add_library.html)
- [Stackoverflow: ExternalProject_Add: BUILD_BYPRODUCTS](https://stackoverflow.com/questions/40314785/linking-against-an-externalproject-add-dependency-in-cmake)
- [GitLab Issue: INTERFACE_INCLUDE_DIRECTORIES does not allow non-existent directories](https://gitlab.kitware.com/cmake/cmake/-/issues/15052)
- [INTERFACE_SYSTEM_INCLUDE_DIRECTORIES](https://cmake.org/cmake/help/latest/prop_tgt/INTERFACE_SYSTEM_INCLUDE_DIRECTORIES.html)
