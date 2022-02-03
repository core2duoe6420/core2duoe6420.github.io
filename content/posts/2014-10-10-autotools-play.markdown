---
title:  "autotools折腾记"
date: 2014-10-10
slug: autotools-play
---

又到了痛并快乐着的折腾时间，这次折腾的对象是autotools。话说很早以前我非常崇拜各种开源代码中的configure和makefile，当初傻傻地以为这都是做项目的人自己写出来的，对开源社区里的人各种膜拜啊，尼玛我看都看不懂不要说写了。后来才知道有autoconf这么个玩意可以自动生成这些脚本，自己当然也要写，但是没有那么夸张了。

<!--more-->

但是这玩意学习成本真心不低啊，首先是中文资料少，我英语废看英文文档累得慌，再加上GNU的手册是字典式的不是教学式的，在零基础的情况下基本上是看不懂的。加上当时没什么动力，所以就没有研究下去了。

## 需求来了

现在动力来了。老师前阵子跟我说现在手上的这个项目，我负责的部分代码要上交，反而是师兄那块技术性更强功能更核心的部分是作为演示的。要上交嘛，就要弄得好看点，正式点。于是我就想了，怎么个正式法？原来那种自己手写的挫的要死的makefile一看就很low，必须要文件夹下一大堆`configue.ac Makefile.am Makefile.in`之类高大上的文件，加上`./configure && make && make install`三步编译才能算得上正式嘛。所以就开始折腾autotools，本着需求第一的原则，搞这个东西我是不求甚解的，真的要完全搞明白，真的太复杂了，只能要用的时候找找资料，学一点，慢慢积累。所以我也真心佩服那些沉得下心，把这些东西都能吃透，样样都精通的大神。

OK，废话不多说，先介绍下需求和大致的环境。

- 代码用到了两个库，`libpcap`和`libdnet`，我向`libdnet`库中添加了一个函数以适应程序要求，所以configure的时候要能检查这个两个库以及新函数是否存在，不存在要报错。
- 代码里用到了DEBUG宏，以DEBUG的值作为调试等级，configure同样要有配置调试级别的选项。
- 因为种种原因，我自己实现了一个哈希表，但也将`glib`中的`GHashTable`适配到了代码里，通过一个宏作为开关，configure的时候要能正确处理宏的定义。

就这三条，其实很简单，autotools的宏已经覆盖了这些功能。程序的目录结构如下：

    --root              根目录
      |--include        头文件.h
      |--src            代码.c
      
## autotools工具和文件
这里的autotools其实指代了一系列工具，要使用autotools得先理清这些工具和它们生成的文件之间的关系。

- `autoconf`：通过`configure.ac`生成`configure`，`configure.ac`是整个流程中最重要的文件。
- `autoheader`：通过`configure.ac`生成`config.h.in`，`config.h.in`是`config.h`的模板文件，`config.h`中包含了整个程序用到的宏定义。
- `autoscan`：扫描源文件进行一些通用的移植配置和宏定义。
- `autoreconf`：以正确的顺序自动运行各种工具，包括`autoconf`、`autoheader`、`automake`。
- `automake`：通过`configure.ac`和`Makefile.in`生成`Makefile`。

还有一些工具更加底层，像这种简单的程序用不到，简单介绍下：

- `autom4te`：是整个autotools工具链的核心，autoconf本质上是M4宏，`autom4te`则是宏处理器。大致上可以理解为类似于C语言编译前的预处理阶段。
- `aclocal`：会生成`aclocal.m4`，包含各种自定义的宏。

实际上，由于我的需求足够简单，在编写完必须的`configure.ac`、`Makefile.am`、`src/Makefile.am`后，只需要执行以下一条命令，就可以生成所有需要的文件了：

```shell
autoreconf --install
```

这就把我从这些复杂的工具中解放了出来，可以专注于核心文件`configure.ac`的编写。

## 编写configure.ac文件
`configure.ac`由一系列的宏组成，经过`autoconf`扩展。这些宏的名字都有约定俗成的规范，比如`AC_`开头的表明是`autoconf`宏，主要用于生成`configure`文件，`AM_`开头的表明是`automake`宏，主要在配合`Makefile.am`生成`Makefile.in`时有效。这里我只介绍会用到的宏。宏里作为参数的部分会用`[]`括起来，这是为了保护参数字符串不被宏处理器继续展开，是M4宏的特性，这里不多做介绍了。

```shell
AC_INIT([convertor],[0.1])
```

初始化`autoconf`，参数是软件名字和版本号，还有一个参数是用于汇报bug的Email地址，我没有提供。这里提供的信息会被写入到`config.h.in`中的`PACKAGE_xxx`宏中。

```shell
AM_INIT_AUTOMAKE([foreign])
```

这句用于初始化`automake`，`autoconf`本身是可以单独使用的，但要生成`Makefile`必须要配合`automake`。这里需要说明的是网上的很多资料里这个宏的参数有三个，与之前的`AC_INIT`参数相同，这是过期的版本，新版本已经不适用了。传入`foreign`的意思是启用foreign模式，工作在GNU模式下的`automake`会要求诸如`NEWS`、`README`、`ChangeLog`这些文件，我没有写这些文件，所以要使用foreign模式取消检查。

```shell
AC_CONFIG_SRCDIR([src/ip.c])
```

这个宏用于检查`configure`所在的位置是否正确，方法就是随便拿一个源文件来测试。

```shell
AC_PROG_CC
```

这个宏用于检查gcc程序，`AC_PROG_xxx`宏用于检查相应的工具是否存在并且工具正常。

```shell
AC_CHECK_LIB([dnet], [eth_send_iovec], [], [AC_MSG_ERROR([libdnet not found or patched.])])
AC_CHECK_LIB([pcap], [pcap_open_live], [], [AC_MSG_ERROR([libpcap not found.])])
```

`AC_CHECK_LIB`宏用于检查程序依赖的库是否存在，方法是检查库中的一个函数能否通过编译，函数名由第二个参数提供，第三个参数是检查成功时执行的代码，最后一个参数是检查失败时执行的代码。这里检查失败时调用`AC_MSG_ERROR`宏，也就是直接报错退出，相应的还有`AC_MSG_WARN`宏，只警告但会继续运行。由于使用一个函数名来检查库是否存在，所以我顺水推舟把我在`libdnet`中新加的函数作为参数提供了，直接就能满足第一个需求。

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

这一段为第二个需求服务。`AC_ARG_ENABLE`宏定义的项可以在运行`./configure`时加上`--enable-xxx[=value]`指定，共有四个参数，第一个是项的名字，第二个是帮助信息，可以通过`./configure --help`查看，第三个是配置时设置了该项时运行的脚本，第四个是没有设置该项时运行的脚本。需要注意的是无论是用`--enable-xxx`还是相应的`--disable-xxx`都会运行第三个选项的脚本，区别只是`$enableval`为`yes`和`no`，当然如果用`=val`设置了值，`$enableval`就会设为`val`。最后是`AC_DEFINE`宏，这个宏会将指定的值写入到`config.h.in`中，对应的格式是：`AC_DEFINE([name],[value],[note])`会转化为：

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

这一段为第三个需求服务。`AC_ARG_WITH`的功能和`AC_ARG_ENABLE`很像，是由`--with-xxx`指定的。参数功能也是一样。`PKG_CHECK_MODULES`宏用于调用`pkg-config`获取库的编译和链接选项，根据参数，执行的功能大致相当于以下两条命令：

```shell
GLIB_CFLAGS=`pkg-config --cflags glib-2.0`
GLIB_LIBS=`pkg-config --libs glib-2.0`
```

`GLIB_CFLAGS`和`GLIB_LIBS`这两个参数会在编写`Makefile.am`时用到。最后`AM_CONDITIONAL`用于设置条件变量，它会判断第二个参数提供的值是否为真，为真在第一个参数的值为`TRUE`，否则`FALSE`，这里定义的值同样是`Makefile.am`用到的。

```shell
AC_CONFIG_HEADERS([config.h])
AC_CONFIG_FILES([Makefile src/Makefile])
AC_OUTPUT
```

最后三句表明输出最终经过`./configure`之后生成的文件。注意每个子目录下都要有Makefile。至此这个`configure.ac`就完成了。

## 编写Makefile.am文件
我们一共要编写两个`Makefile.am`文件。第一个位于根目录下，很简单，就一句话：

```shell
SUBDIRS = src
```

指定要递归进入的子目录。这里只有一项，当然可以指定许多项，空格隔开，且每个子目录下都要有`Makefile.am`文件。

第二个`Makefile.am`文件自然是在`src`文件夹下。

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

这段代码还是很简洁易懂的，需要说明的有几点：

- `bin_PROGRAMS`中的`bin`说明程序最终通过`make install`安装在`${exec_prefix}/bin`下。`${exec_prefix}`是`autoconf`预设的变量，默认值等于`${prefix}`，还有许多类似的项，如`lib`指明的会安装在`${exec_prefix}/lib`下，`noinst`则表明不安装。`PROGRAMS`则表明生成的可执行文件，类似的还有`LIBRARIES`表明库文件，`HEADERS`表明是头文件，等等，因为我这里用不到，也没有做其他的实验。
- 通过第一项指定目标名字后，就可以通过`name_CFLAGS`、`name_LDFLAGS`、`name_SOURCES`等设置程序专属的变量，这些变量有的有相应的公共值由所有目标共享，例如`AM_CFALGS`、`AM_LDFLAGS`。
- `LDADD`专门用于添加所需的库，由`LDADD`指定的库会被放在gcc链接命令最后，确保正确链接（如果写在`LDFLAGS`里会被写在gcc链接命令`.o`文件的前面导致无法正确链接），`LDADD`变量的公共版就叫`LDADD`，不用添加`AM_`前缀。另外，在`configure.ac`中通过`AC_CHECK_LIB`宏检测的库会自动添加到编译命令中，所以不需要单独设置，可以看到代码中那行被我注释了。
- 这里也能看到`configure.ac`中指定的`USE_GLIB`、`GLIB_CFLAGS`、`GLIB_LIBS`的用法了，很清晰就不多说了。

## 参考资料

编写完两个文件就能使用`autoreconf --install`命令生成所需的所有文件了，然后就能像最常见的程序那样`./configure && make && make install`编译安装了。

最后给出我折腾过程中参考的资料。

- 首先是我感觉最有价值的入门材料，做成了PPT的样子，不过通过PDF发布，一共是162张PPT，不过他把每一步动画都分隔到了一页上导致PDF显示有556页，刚开始看的时候吓尿了，后来发现其实还好。我基本上就是根着这个PDF学习的。这是[下载地址](https://www.lrde.epita.fr/~adl/autotools.html)。
- [Autoconf Manual](http://www.gnu.org/software/autoconf/manual/autoconf.html)总是逃不掉的。
- [《AUTOTOOLS: A PRACTITIONER'S GUIDE TO GNU AUTOCONF, AUTOMAKE, AND LIBTOOL》](http://dl.e-book-free.com/2013/07/autotools.pdf)，这个就比较狠了，是一本书，三百多页，当然也相对系统。
- 网上各种博客，比如[这个](http://blog.csdn.net/scucj/article/details/6079052)，不得不说autotools的中文资料很少，网上很多还是过期的，要想学习还得看英文，好痛苦。