---
title: "Merge Multiple Static Libraries into One"
date: 2023-09-19T18:16:45+08:00
draft: true
---

## Multiple Static -> Static

```
MSVC: lib.exe
Linux: ar -rcT libgaea.a libgaea_base.a libgaea_config.a
Apple: libtool

```

## Multiple Static -> Shared


There was a CMake snippet which is a cross-platform solution on gist. Unfortunately cannot be found again.

```
MSVC: /WHOLE_ARCHIVE
gcc: --whole-archive

```
