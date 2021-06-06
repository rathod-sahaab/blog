---
title: Accelerating libnbd on linux using io_uring [2] - Project Setup
date: '2021-06-06T11:58+00:00'
template: 'post'
draft: false
slug: 'libnbd-io_uring-2'
category: 'Systems Programming'
tags:
  - 'libnbd'
  - 'iouring'
  - 'systems-programming'
description: 'io_uring is the new and better way to do asynchronus I/O on linux, tag along with me on a journey to learn it by doing, on a real life project libnbd!'
socialImage: '/media/io_uring-sq-cq.png'
---

Coding Begins
---
6 June 2021

To be able to use io\_uring we should be able to `#include <liburing.h>`, well we can do that anyways but we have to make sure it includes desired libraries.

## Project Setup
`libnbd` uses [autotools](https://www.gnu.org/software/automake/manual/html_node/Autotools-Introduction.html) a set `autoconf`, `automake`, etc. Main files you edit are `configure.ac` (ac: autoconf), `Makefile.am` (am: automake).

We need to change `configure.ac` and add `liburing` as dependency/requirement. Now, I am no *autotools* wizard, my mentor [Richard](https://rwmj.wordpress.com/) is, so I took the easy road and copied his code and modified it to my need after browsing the documentation and verifying it does what I want :)

Original code for `libxml2`
```m4
dnl Check for libxml2 (optional, for NBD URI support)
AC_ARG_WITH([libxml2],
    [AS_HELP_STRING([--without-libxml2],
                    [disable use of libxml2 for URI support @<:@default=check@:>@])],
    [],
    [with_libxml2=check])
AS_IF([test "$with_libxml2" != "no"],[
    PKG_CHECK_MODULES([LIBXML2], [libxml-2.0], [
        AC_SUBST([LIBXML2_CFLAGS])
        AC_SUBST([LIBXML2_LIBS])
        AC_DEFINE([HAVE_LIBXML2],[1],[libxml2 found at compile time.])
    ], [
        AC_MSG_WARN([libxml2 not found, NBD URI support will be disabled.])
    ])
])
AM_CONDITIONAL([HAVE_LIBXML2], [test "x$LIBXML2_LIBS" != "x"])
```

I modified it to code below for `liburing`
```m4
dnl Check for liburing (optional, for io_uring support)
AC_ARG_WITH([liburing],
    [AS_HELP_STRING([--without-liburing],
                    [disable use of liburing i.e. io_uring for asynchronus io @<:@default=check@:>@])],
    [],
    [with_liburing=check])
AS_IF([test "$with_liburing" != "no"],[
    PKG_CHECK_MODULES([LIBURING], [liburing], [
        AC_SUBST([LIBURING_CFLAGS])
        AC_SUBST([LIBURING_LIBS])
        AC_DEFINE([HAVE_LIBURING],[1],[liburing found at compile time.])
    ], [
        AC_MSG_WARN([liburing not found, io_uring support disabled.])
    ])
])
AM_CONDITIONAL([HAVE_LIBURING], [test "x$LIBURING_LIBS" != "x"])
```

Then I ran some commands to generate `Makefile`
```bash
autoreconf -i
./configure
```

To make/compile
```bash
make
```

### Linting

I like my linter to check for bugs while I edit the code and get auto completion. The linter `coc-clangd` uses `compile_commands.json` which autotools don't generate as far as I know. **VS Code** also uses `compile_commands.json`.

To generate compile commands I use **bear** everytime I change `configure.ac`
```
bear -- make
```

Then on subsequent code changes only 
```
make
```

