---
title:  "autotools Tinkering Notes"
date: 2014-10-10
slug: autotools-play
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

It’s time again for that painful-yet-joyful tinkering, and this time the target is autotools. Back in the day I used to admire the `configure` and `Makefile` files in various open-source projects. I naively thought they were all handwritten by the project authors themselves, and I was full of awe for people in the open-source community. I couldn’t even understand them, let alone write them. Later I learned there is something called `autoconf` that can automatically generate these scripts. You still have to write some bits yourself, but it’s nowhere near as crazy as I had imagined.

<!--more-->

However, the learning cost of this thing is really not low. First, there’s little Chinese documentation, and my poor English makes reading English docs exhausting. On top of that, the GNU manuals are like dictionaries rather than tutorials; if you’re starting from zero it’s basically impossible to understand them. I also didn’t have much motivation back then, so I never really dug into it.

## Requirements show up

Now the motivation is here. Some time ago my advisor told me that for the current project, the part I’m responsible for needs to be submitted, while the more technically challenging and core part by my senior will be used only for demonstration. Since it’s going to be submitted, it has to look decent and “formal”. So I started thinking: what counts as “formal”? Those crappy hand-written Makefiles I had before look low-end at first glance. Obviously I needed a big pile of “fancy” files in the directory like `configue.ac Makefile.am Makefile.in`, and only with the classic three-step build `./configure && make && make install` could it be considered formal. So I started messing with autotools, and sticking to the “requirement-first” principle, I don’t intend to fully understand it. Really mastering this stuff is too complicated; I can only look things up when I need them, learn a bit at a time, and accumulate slowly. I genuinely admire those who can sit down and digest all of this, mastering every piece.

OK, enough chatter, let’s talk about the requirements and the general environment.

- The code uses two libraries, `libpcap` and `libdnet`. I added a function to the `libdnet` library to satisfy the program’s needs, so during configure we must check for the presence of both libraries and the new function, and report an error if they are missing.
- The code uses a DEBUG macro, with the value of DEBUG indicating the debug level. `configure` must provide an option to configure the debug level.
- For various reasons I implemented my own hash table, but I also adapted `glib`’s `GHashTable` into the code, controlled via a macro switch. `configure` needs to correctly handle the macro definition.

Just these three items, which are actually quite simple, and autotools macros already cover these features. The program’s directory structure is as follows:

```text
--root              root directory
  |--include        header files .h
  |--src            source files .c
```

## autotools tools and files

Here “autotools” actually refers to a set of tools. To use autotools, you have to first clarify the relationships between these tools and the files they generate.

- `autoconf`: generates `configure` from `configure.ac`. `configure.ac` is the most important file in the whole process.
- `autoheader`: generates `config.h.in` from `configure.ac`. `config.h.in` is the template for `config.h`; `config.h` contains all the macro definitions used by the program.
- `autoscan`: scans the source files and suggests some common portability checks and macro definitions.
- `autoreconf`: runs the various tools in the correct order, including `autoconf`, `autoheader`, `automake`.
- `automake`: generates `Makefile` from `configure.ac` and `Makefile.in`.

There are also some lower-level tools that simple programs like this don’t need. Briefly:

- `autom4te`: this is the core of the entire autotools toolchain. `autoconf` is essentially a set of M4 macros, and `autom4te` is the macro processor. Roughly, you can think of it as something like the preprocessing stage before C compilation.
- `aclocal`: generates `aclocal.m4`, which contains various custom macros.

In fact, because my requirements are simple enough, after writing the necessary `configure.ac`, `Makefile.am`, and `src/Makefile.am`, I only need to run this one command to generate all required files:

```shell
autoreconf --install
```

This frees me from the complexity of these tools so I can focus on the core file `configure.ac`.

## Writing configure.ac

`configure.ac` consists of a series of macros that are expanded by `autoconf`. These macro names follow conventional rules: those starting with `AC_` are `autoconf` macros, mainly used to generate the `configure` file; those starting with `AM_` are `automake` macros, mainly effective when generating `Makefile.in` together with `Makefile.am`. I’ll only introduce the macros I use. Macro parameters are wrapped in `[]`, which protects the argument strings from further expansion by the macro processor; this is a feature of M4 macros and I won’t go into detail here.

```shell
AC_INIT([convertor],[0.1])
```

Initializes `autoconf`. The arguments are the software name and version number; there’s another parameter for the bug-reporting email address, which I did not provide. The information provided here will be written into `config.h.in` as `PACKAGE_xxx` macros.

```shell
AM_INIT_AUTOMAKE([foreign])
```

This initializes `automake`. `autoconf` can be used on its own, but to generate `Makefile` you must use it together with `automake`. One thing to note is that many online resources show this macro with three parameters, matching the earlier arguments of `AC_INIT`; that’s the old version and no longer applies to newer versions. Passing `foreign` enables the “foreign” mode. In GNU mode, `automake` requires files such as `NEWS`, `README`, `ChangeLog`, etc. Since I don’t have these files, I use foreign mode to disable those checks.

```shell
AC_CONFIG_SRCDIR([src/ip.c])
```

This macro checks whether `configure` is being run in the correct directory by testing the existence of some source file.

```shell
AC_PROG_CC
```

This macro checks for the `gcc` program. `AC_PROG_xxx` macros are used to check whether the corresponding tools exist and work properly.

```shell
AC_CHECK_LIB([dnet], [eth_send_iovec], [], [AC_MSG_ERROR([libdnet not found or patched.])])
AC_CHECK_LIB([pcap], [pcap_open_live], [], [AC_MSG_ERROR([libpcap not found.])])
```

The `AC_CHECK_LIB` macro checks whether a dependency library exists by trying to compile against a function from that library. The function name is given by the second parameter. The third parameter is shell code to execute if the check succeeds, and the last parameter is shell code if the check fails. On failure I call the `AC_MSG_ERROR` macro, which prints an error and exits. There is also `AC_MSG_WARN`, which only prints a warning and continues. Since a function name is used to test for the presence of a library, I simply used the function I added to `libdnet` as the argument, which directly satisfies the first requirement.

```shell
AC_ARG_ENABLE([debug],
    [AS_HELP_STRING([--enable-debug[=level]],[运行时打印调试信息（默认关闭）])],
    [
        if test "x$enableval" = "xyes"; then
            AC_DEFINE([DEBUG],[2],[Define if --enable-debug])
        elif test "x$enableval" = "x0" ; then
            AC_DEFINE([DEBUG],[0],[Define if --enable-debug])
        elif test "x$enableval" = "x1" ; then
            AC_DEFINE([DEBUG],[1],[Define if --enable-debug])
        elif test "x$enableval" = "x2" ; then
            AC_DEFINE([DEBUG],[2],[Define if --enable-debug])
        elif test "x$enableval" = "x3" ; then
            AC_DEFINE([DEBUG],[3],[Define if --enable-debug])
        else
            echo "Error! Unknown DEBUG level"
            exit -1
        fi
    ],
    []
)
```

This part serves the second requirement. Options defined by `AC_ARG_ENABLE` can be set when running `./configure` using `--enable-xxx[=value]`. It has four parameters: the first is the option name; the second is the help string, which can be viewed with `./configure --help`; the third is the script executed when this option is specified; the fourth is the script executed when the option is not specified. Note that both `--enable-xxx` and the corresponding `--disable-xxx` will run the third script; the difference is that `$enableval` is `yes` or `no`. If a `=val` is provided, then `$enableval` is set to `val`. Finally, the `AC_DEFINE` macro writes the specified value into `config.h.in`, in the form `AC_DEFINE([name],[value],[note])`, which becomes:

```c
/* note */
#define name value
```

```shell
AC_ARG_WITH([glib],
    [AS_HELP_STRING([--with-glib], [使用glib中的HashTable（默认关闭）])],
    [  
        if test "x$withval" = "xyes"; then
            AC_DEFINE([_USING_GLIB_HASHTABLE],[1],[Define if --with-glib])
            PKG_CHECK_MODULES([GLIB], [glib-2.0])
            with_glib="yes"
        fi
    ],
    []
)
AM_CONDITIONAL([USE_GLIB], [test "x$with_glib" = "xyes"])
```

This part serves the third requirement. `AC_ARG_WITH` is very similar to `AC_ARG_ENABLE`, but is specified via `--with-xxx`. The parameters work the same way. The `PKG_CHECK_MODULES` macro uses `pkg-config` to obtain compilation and linking options for a library. According to the arguments, its effect is roughly equivalent to:

```shell
GLIB_CFLAGS=`pkg-config --cflags glib-2.0`
GLIB_LIBS=`pkg-config --libs glib-2.0`
```

The two variables `GLIB_CFLAGS` and `GLIB_LIBS` will be used when writing `Makefile.am`. Finally, `AM_CONDITIONAL` sets a conditional variable: it evaluates the second argument and, if true, sets the first argument to `TRUE`, and to `FALSE` otherwise. The value defined here is also used in `Makefile.am`.

```shell
AC_CONFIG_HEADERS([config.h])
AC_CONFIG_FILES([Makefile src/Makefile])
AC_OUTPUT
```

These last three lines specify the output files generated after running `./configure`. Note that every subdirectory must have a Makefile. At this point, `configure.ac` is done.

## Writing Makefile.am

We need to write two `Makefile.am` files. The first one is in the root directory and is very simple, just one line:

```shell
SUBDIRS = src
```

This specifies which subdirectories to recurse into. Here there is only one, but you can specify many, separated by spaces, and each subdirectory must have its own `Makefile.am`.

The second `Makefile.am` is of course in the `src` directory.

```shell
bin_PROGRAMS = convertor
convertor_CFLAGS = -std=gnu99 -I$(srcdir)/../include -O
if USE_GLIB
    convertor_CFLAGS += $(GLIB_CFLAGS)
endif
convertor_LDFLAGS = -pthread
if USE_GLIB
    convertor_LDADD = $(GLIB_LIBS)
endif
#convertor_LDADD = -ldnet -lpcap
convertor_SOURCES = a.c b.c c.c
```

This piece is quite concise and easy to understand, but a few points are worth noting:

- In `bin_PROGRAMS`, `bin` means the program will be installed under `${exec_prefix}/bin` by `make install`. `${exec_prefix}` is a variable preset by `autoconf`; by default it equals `${prefix}`. There are many similar ones: for example, with `lib` the files will be installed under `${exec_prefix}/lib`, while `noinst` means they are not installed. `PROGRAMS` indicates the target is an executable; similarly there are `LIBRARIES` for libraries, `HEADERS` for header files, and so on. I don’t use these here and haven’t experimented further.
- After specifying the target name with the first line, you can use `name_CFLAGS`, `name_LDFLAGS`, `name_SOURCES`, etc. to set per-target variables. Some of these have corresponding global values shared by all targets, such as `AM_CFLAGS`, `AM_LDFLAGS`.
- `LDADD` is specifically used to add the required libraries. Libraries specified in `LDADD` are placed at the end of the gcc link command to ensure correct linking (if they are placed in `LDFLAGS`, they may appear before the `.o` files in the gcc link command and fail to link correctly). The global version of `LDADD` is just `LDADD` (without the `AM_` prefix). In addition, libraries checked via `AC_CHECK_LIB` in `configure.ac` are automatically added to the compile command, so they don’t need to be set separately; you can see that the line in the code is commented out.
- Here you can also see how `USE_GLIB`, `GLIB_CFLAGS`, and `GLIB_LIBS` defined in `configure.ac` are used; it’s quite straightforward, so I won’t say more.

## References

Once the two files are written, you can run `autoreconf --install` to generate all necessary files, and then build and install like the most common programs using `./configure && make && make install`.

Finally, here are the materials I referenced during my tinkering:

- First is what I consider the most valuable introductory material. It’s like a set of PPT slides but published as a PDF. There are 162 PPT slides, but each animation step is split into a separate page, so the PDF shows 556 pages. It scared me at first, but it turned out to be fine. I basically learned by following this PDF. Here is the [download link](https://www.lrde.epita.fr/~adl/autotools.html).
- The [Autoconf Manual](http://www.gnu.org/software/autoconf/manual/autoconf.html) is unavoidable.
- [“AUTOTOOLS: A PRACTITIONER'S GUIDE TO GNU AUTOCONF, AUTOMAKE, AND LIBTOOL”](http://dl.e-book-free.com/2013/07/autotools.pdf) is more serious: a three-hundred-plus-page book, but more systematic.
- Various blogs on the web, for example [this one](http://blog.csdn.net/scucj/article/details/6079052). Autotools documentation in Chinese is really scarce, and much of what’s online is outdated. To really learn it, you have to read English, which is quite painful.