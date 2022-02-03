---
title: "交叉编译ARM Native GCC"
date: 2014-09-10
slug: cross-compile-arm-natvie-gcc
---

前阵子学会了用ip tunnel建立隧道来让没有原生IPv6的电脑用上IPv6，心里痒痒想在家里也提前用上IPv6，于是狠下心买了心仪已久的R6300v2，配上ddwrt，顺利用上了IPv6。

<!--more-->

R6300v2配备了BCM4708，ARM cortex-a9双核800Mhz，最新的ddwrt的内核版本是3.10.25，可以说非常新了，于是就想能不能在路由器上面跑GCC，编译程序。在网上找了好久，能在路由器上直接运行的GCC找不到，能交叉编译出路由器能运行的程序的GCC倒是有，openwrt那边一大堆。我没有交叉编译的真实经历，但是寒假的时候看过一本《深度探索Linux操作系统 系统构建和原理解析》（以下简称《简析》），当初跟著书上的步骤一步步做下来的，虽然成功了但没怎么搞懂，于是萌发了自己从头构建一个能在路由器上运行的GCC的想法，就此开始了长达四天的折腾之旅。

## 环境

宿主机：`Ubuntu 14.04 64位，gcc 4.8.2`

工具链版本：`binutils-2.24 gcc-4.9.1 glibc-2.19 linux-3.10.25`（因为我安装的ddwrt内核是3.10.25）

编译gcc需要三个额外的库，gmp mpc mpfr，从网上下载后解压，把文件夹名从gmp-version mpc-version mpfr-version改为gmp mpc mpfr，然后放到gcc-4.9.1文件夹下，gcc较新版本可以正确处理这几个库的位置了，不用其他参数指定（gcc-4.7.4需要configure的时候指定`--with-mpfr-include`和`--with-mpfr-lib`，折腾的时候得到的教训）。

首先交叉编译要确定目标机器的三元组，也就是arch-vendor-os，在Ubuntu中这个三元组是`x86_64-pc-linux-gnu`，而R6300v2上的三元组，经过一番折腾后最终我用的是`arm-ddwrt-linux-gnueabi`。这个三元组非常重要，因为事关C库的编译。网上经常能见到的有`arm-none-eabi`，这个表示没有操作系统，使用newlib作为C库，折腾的时候我也试过这个三元组，后来发现没有操作系统怎么可能是我要的。还有最后的`gnueabi`，这个还有变种`gnueabihf`，表示CPU有FPU，hf的意思据我推测应该是hard float，但是据说因为路由器不需要浮点运算，所以R6300v2把FPU阉割掉了-:(……最后还有一种也就是我网上找到的交叉编译器`arm-openwrt-linux-uclibcgnueabi`，其实是使用uclibc作为C库的编译器，ddwrt也自带了uclibc，所以用它编译的程序是可以在R6300v2上运行的。另外也可以看出vendor其实无所谓，随便取好了……

全部源代码解压完后，需要对gcc做个patch。因为我的情况比较特殊，路由器上的/挂载位置是只读的，意味着我没法往常规目录/lib和/usr写入文件，况且路由器的内置容量也放不下gcc和glibc这样的庞然大物，好在R6300v2有USB接口，可以插上USB存储设备，我把U盘挂载在`/tmp/mnt/usb`下（因为/tmp是ramfs，可写），把glibc的链接文件都放在`/tmp/mnt/usb/lib`下，把/usr下的文件放到`/tmp/mnt/usb/usr`下。但是程序启动时操作系统加载的dynamic linker是gcc硬编码在代码里的，所以要修改dynamic linker的位置。

关于dynamic linker，可以使用`readelf -l a.out`查看，输出中会有一句

```no-highlight
[Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

这是我在Ubuntu x86_64上的输出结果，而我的目标是

```no-highlight
[Requesting program interpreter: /tmp/mnt/usb/lib/ld-linux.so.3]
```

在gcc-4.9.1目录下执行以下代码，即可打上补丁，代码来自LFS的教程，我加了一项linux-eabi.h，因为我的目标是ARM而LFS的目标是x86。

```no-highlight
for file in $(find gcc/config -name linux64.h -o -name linux.h -o -name sysv4.h -o -name linux-eabi.h)
do
    cp -uv $file{,.orig}
    sed -e 's@/lib\(64\)\?\(32\)\?/ld@/tmp/mnt/usb&@g' \
    -e 's@/usr@/tmp/mnt/usb@g' $file.orig > $file
    echo '
#undef STANDARD_STARTFILE_PREFIX_1
#undef STANDARD_STARTFILE_PREFIX_2
#define STANDARD_STARTFILE_PREFIX_1 "/tmp/mnt/usb/lib/"
#define STANDARD_STARTFILE_PREFIX_2 ""' >> $file
    touch $file.orig
done
```

## 环境变量

这些是我在命令中用到的环境变量，来源上面都说明过了。BUILD\_DIR就是编译的工作目录。要说明的是CROSS\_TOOL目录下放入的在x86上运行的交叉编译工具，而SYSROOT目录下放的是在R6300v2上运行的程序，包括最终生成gcc、glibc库和头文件。交叉工具链编译时的sysroot机制太复杂了，我也没有完全搞清……至于为什么是/vita，因为一开始我是跟着《简析》来做的，书里面用的是/vita。

```no-highlight
THREADS=2
TARGET=arm-ddwrt-linux-gnueabi
HOST=x86_64-pc-linux-gnu
BUILD=$HOST
INSTALL_ROOT=/vita
SYSROOT=$INSTALL_ROOT/sysroot
CROSS_TOOL=$INSTALL_ROOT/cross-tool
HOST_ROOT_PREFIX=/tmp/mnt
HOST_ROOT=$HOST_ROOT_PREFIX/usb
BUILD_DIR=$INSTALL_ROOT/build
```

## binutils round 1

交叉编译的第一步永远是binutils，目标是编译出在x86上运行的目标代码为arm的ld、as等。脚本：

```no-highlight
cd binutils-build
../binutils-2.24/configure                          \
    --prefix=$CROSS_TOOL                            \
    --with-sysroot=$SYSROOT                         \
    --target=$TARGET                                \
    --disable-nls                                   \
    --disable-libssp
make -j $THREADS
make install
```

参数说明（所有的参数说明都是我根据自己的理解写的，不一定对，甚至错误的概率很高，好多我也只是根据网上抄的，但能用。）：

- `--disable-nls`：nls的意思是Native Language Support，就是说编译期间只用英语提示。
- `--disable-libssp`：ssp好像是栈保护什么的，我也不知道为什么要禁用，反正大家都禁用了……

## gcc round 1

第一次编译gcc的目标是在没有C库的支持下编译出一个能用的最小功能的gcc，然后用它来编译glibc，关键的几个参数是`--without-headers`和`--with-newlib`，这两个参数告诉gcc用自带的头文件。脚本：

```no-highlight
cd ../gcc-build
../gcc-4.9.1/configure                              \
    --target=$TARGET                                \
    --prefix=$CROSS_TOOL                            \
    --with-sysroot=$SYSROOT                         \
    --with-newlib                                   \
    --without-headers                               \
    --disable-nls                                   \
    --disable-shared                                \
    --disable-multilib                              \
    --disable-decimal-float                         \
    --disable-threads                               \
    --disable-libatomic                             \
    --disable-libgomp                               \
    --disable-libitm                                \
    --disable-libquadmath                           \
    --disable-libsanitizer                          \
    --disable-libssp                                \
    --disable-libmudflap                            \
    --disable-libvtv                                \
    --disable-libcilkrts                            \
    --disable-libstdc++-v3                          \
    --enable-languages=c                            \
    --with-float=soft                               \
    --with-local-prefix=$HOST_ROOT/usr/local        \
    --with-native-system-header-dir=$HOST_ROOT/usr/include
make -j $THREADS
make install
```

参数说明：

- `--target`：当target参数host参数不一样时，启用交叉编译。host没有指定时gcc会自动判断为当前的三元组。
- `--prefix`：这个参数只影响最终gcc的安装位置。
- `--disable-xxx`：第一次编译时禁用大部分的功能，因为没有C库编译不了。
- `--with-float=soft`：gcc配置时的`--with-xxx`参数大多数是用来指明编译出来的gcc在编译其他程序时的默认选项，同样的还有`--with-cpu --with-arch`等，使用了这个参数，当用编译出来的gcc编译其他程序的时候，会自动带上`-msoft-float`（不知道理解的对不对，可以参考[Installing GCC: Configuration](https://gcc.gnu.org/install/configure.html)和[ARM Options](https://gcc.gnu.org/onlinedocs/gcc-4.5.3/gcc/ARM-Options.html))。
- `--with-local-prefix`：这个参数会把目录放到gcc默认的头文件搜索路径中，默认是`/usr/local`，参数中不需要`include`，但实际上是到`/usr/local/include`中找头文件的，这里之所以要改当然也是因为上一节中说明的问题。
- `--with-native-system-header-dir`：这个参数在第一轮编译gcc时其实用不到，要第二轮开始才用。它配合`--with-sysroot`指定在**编译期间**以及编译出来的gcc查找头文件的位置，所以最终查找头文件的位置是`$SYSROOT/$HOST_ROOT/usr/include`。为此我们需要把linux内核头文件和glibc的头文件都安装到这个位置。

## 内核头文件

安装内核头文件比较简单，要注意的是ARCH参数，以及安装位置，这在上面介绍gcc参数时已经说过原因了。
```no-highlight
cd ../linux-3.0.25
make mrproper
make ARCH=arm INSTALL_HDR_PATH=$SYSROOT/$HOST_ROOT/usr headers_install
```

## glibc

有了最初的gcc后，就可以用它来编译glibc了，glibc的配置中`--host`表明我们要生成在目标机器上运行的glibc，所以生成出来的glibc是不能在x86 Ubuntu上直接运行的。glibc也没有target的概念。

```no-highlight
cd ../glibc-build
rm -rf *
../glibc-2.19/configure                             \
    --prefix=$HOST_ROOT                             \
    --host=$TARGET                                  \
    --disable-profile                               \
    --enable-kernel=2.6.32                          \
    --with-headers=$SYSROOT/$HOST_ROOT/usr/include  \
    libc_cv_forced_unwind=yes                       \
    libc_cv_ctors_header=yes                        \
    libc_cv_c_cleanup=yes
make -j $THREADS
make install_root=$SYSROOT install
```

参数说明：

- `--prefix`：指定安装目录。glibc的安装脚本也会把这个参数写入库文件的依赖中，在目标系统上库文件会到相应目录查找动态库，所以这里没有直接加上$SYSROOT，而是在安装的时候才指定安装目录为$SYSROOT。
- `--with-headers`：指定linux内核头文件目录，就是上一步中安装的位置。
- `libc_cv_*`：这三个是用来欺骗glibc的，具体的请参考[Linux From Scratch 5.7 Glibc-2.19](http://www.linuxfromscratch.org/lfs/view/development/chapter05/glibc.html)。

glibc安装完成后，默认把头文件放在`$SYSROOT/$HOST_ROOT/include`下，根据上面说的，我们要把它移动到`$SYSROOT/$HOST_ROOT/usr/include`下：

```no-highlight
cp -r $SYSROOT/$HOST_ROOT/include $SYSROOT/$HOST_ROOT/usr/
```

## gcc round 2

有了glibc之后就可以重新编译gcc，并开启大部分功能了。

```no-highlight
cd ../gcc-build
rm -rf *
../gcc-4.9.1/configure                              \
    --target=$TARGET                                \
    --prefix=$CROSS_TOOL                            \
    --with-sysroot=$SYSROOT                         \
    --disable-nls                                   \
    --enable-shared                                 \
    --disable-multilib                              \
    --enable-threads=posix                          \
    --enable-languages=c,c++                        \
    --disable-libssp                                \
    --disable-decimal-float                         \
    --disable-libgomp                               \
    --with-float=soft                               \
    --with-local-prefix=$HOST_ROOT/usr/local        \
    --with-native-system-header-dir=$HOST_ROOT/usr/include
make -j $THREADS
make install
```

参数说明：

- `--disable-decimal-float`：一次编译时提示目标系统不支持decimal-float，所以把它禁掉。
- `--disable-multilib --disable-libgomp`：这两个不加会导致编译出错（libiberty和zlib报错，具体原因也不清楚）。
- `--with-native-system-header-dir`：这个参数现在起作用了。

这一轮编译完后，作为交叉编译工具的gcc已经完成了，用它编译出来的二进制文件，应该可以正常在R6300v2上运行了（当然前提是glibc的库文件放在正确的位置，也就是`/tmp/mnt/usb/lib`下）。写一个hello world，用readelf读取下elf的头信息：

```no-highlight
vita@cc-virtual-machine:/vita$ arm-ddwrt-linux-gnueabi-gcc a.c
vita@cc-virtual-machine:/vita$ ./a.out
-su: ./a.out: cannot execute binary file: Exec format error
vita@cc-virtual-machine:/vita$ readelf -h a.out
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x82d0
  Start of program headers:          52 (bytes into file)
  Start of section headers:          4444 (bytes into file)
  Flags:                             0x5000202, has entry point, Version5 EABI, soft-float ABI
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         8
  Size of section headers:           40 (bytes)
  Number of section headers:         37
  Section header string table index: 34
```

## binutils round 2

第二轮编译binutils的目标就是编译出能在R6300v2上运行的ld和as这些程序。关键参数是`--host`，指定了该参数后makefile就会调用`$TARGET-gcc`，也就是`arm-ddwrt-linux-gnueabi-gcc`等一系列交叉编译工具来编译binutils，而不是Ubuntu自带的。
```no-highlight
cd ../binutils-build
rm -rf *
../binutils-2.24/configure                          \
    --prefix=$SYSROOT/$HOST_ROOT/usr                \
    --with-sysroot                                  \
    --host=$TARGET                                  \
    --disable-nls                                   \
    --disable-libssp                                \
    --disable-werror
make -j $THREADS
make install
```

参数说明：

- `--prefix`：因为要编译出在R6300v2上运行的binutils，所以安装到目标文件夹下。
- `--disable-werror`：不加这个参数编译期间会因为将警告视为错误而终止。

## gcc round 3

第三轮gcc目标当然是编译出能运行在R6300v2上的gcc编译器了。在编译开始前要先把`$SYSROOT/$HOST_ROOT/usr/include`下的文件拷贝到Ubuntu下的`$HOST_ROOT/usr/include`，也就是`/tmp/mnt/usb/usr/include`下，因为编译出的gcc要在R6300v2上运行，配置时的`--with-sysroot`参数不能想之前两轮那样设置为`$SYSROOT`了，而必须设置为`/`。所以头文件必须放到由`--with-native-system-header-dir`参数直接指定的目录下：

```no-highlight
rm -rf $HOST_ROOT_PREFIX
mkdir -p $HOST_ROOT/usr
cp -r $SYSROOT/$HOST_ROOT/usr/include $HOST_ROOT/usr

cd ../gcc-build
rm -rf *
../gcc-4.9.1/configure                              \
    --build=x86_64-pc-linux-gnu                     \
    --target=$TARGET                                \
    --host=$TARGET                                  \
    --with-sysroot=/                                \
    --prefix=$SYSROOT/$HOST_ROOT/usr                \
    --disable-nls                                   \
    --enable-shared                                 \
    --disable-multilib                              \
    --disable-decimal-float                         \
    --enable-threads=posix                          \
    --disable-libgomp                               \
    --disable-libssp                                \
    --enable-languages=c,c++                        \
    --enable-__cxa_atexit                           \
    --with-float=soft                               \
    --disable-libstdcxx-pch                         \
    --disable-bootstrap                             \
    --with-local-prefix=$HOST_ROOT/usr/local        \
    --with-native-system-header-dir=$HOST_ROOT/usr/include 
make -j $THREADS
make install
```

参数说明：

- `--with-sysroot`：因为目标是能在R6300v2上运行的gcc，所以不能像之前那样把sysroot设置成`$SYSROOT`了，这样会使gcc在R6300v2上找错头文件和库的位置。
- `--disable-bootstrap`：gcc编译时有一个机制是用编译出来的gcc再编译一次自身看看能不能成功编译，这里如果开始这个机制会报错，具体原因不明，还是参考[Installing GCC: Configuration](https://gcc.gnu.org/install/configure.html)。

## 结束语

至此gcc成功在R6300v2上运行了，但还不是很完美，用g++编译出来的程序结束时会出现段错误。原因我还不清楚，现在也不想深究了，毕竟gcc没有什么问题，也算是完成了最初的目标。但是要把路由器打造成一个真实的编译环境显然还差的很远，缺少make，缺少各种库，而且也不现实，毕竟CPU没有那么强悍，还不如用交叉编译工具链搞完后直接放上去。

在整整四天的折腾过程中，我编译了二十来遍gcc，家里的奔腾G2020太弱，编译一次要半小时以上，可以说弄得身心俱疲，全凭着一股怨念支撑着我，也让我深深感受到一点，没有理论只是支撑的瞎折腾纯属浪费时间。所以还是好好学习吧。

## 后记

昨天写完这篇文章后，我兴致冲冲的想用编译出来的native gcc编译自己以前写的程序，也算是测试，要是连我自己写的小程序都编译不了，那不是扯淡么。我在以前写过的程序里挑了一个不依赖其他库的，就是大三搞的那个PL0，开始编译。第一步先编译在R6300v2上的make。

#### make

我下载的make版本是3.8.2，最新版好像是4.0.0了。make的编译很简单。把它安装在目标系统的位置下就可以了。
```no-highlight
./configure --host=$TARGET --prefix=$SYSROOT/$HOST_ROOT/usr/
make -j $THREADS
make install
```
make编译完成后，我就直接把PL0代码拷到路由器上开始跑了。编译成功！我很开心，运行，`segmentation fault`！我很纳闷，程序在x86上跑的很正常，是不是gcc啥的又有问题。简单加了几句printf定位问题，发现是在词法分析的地方，代码太久远了，我自己也不清楚里面的结构了，就不要说改了。一狠心我就想再编译个gdb调试，于是又折腾了一阵子。

#### gdb

gdb依赖termcap，先从网上下载`termcap-1.3.1.tar.gz`，这是一个非常古老的程序，编译，安装。

```no-highlight
./configure --prefix=/vita/termcap --host=$TARGET
make
make install
```

termcap编译出来的是静态链接文件`libtermcap.a`,所以我偷懒没有把它装到目标系统位置下，随便找了个目录放。

termcap编译后就可以编译gdb了。先用几个环境变量指定termcap的位置。

```no-highlight
export LDFLAGS="-static -L/vita/termcap/lib"
export CPPFLAGS="-I/vita/termcap/include"
export CFLAGS="-I/vita/termcap/include"
```

然后正常编译就可以了。

```no-highlight
../gdb-7.8/configure --host=$TARGET --prefix=$SYSROOT/$HOST_ROOT/usr/
make -j $THREADS
make install
```

###问题所在

大功告成，开始用gdb调试，发现问题出在一个while循环里，看样子是死循环下标越界导致的段错误。可是这段代码在x86上跑的好好的，应该不是算法问题才对。因为循环不跳出所以看了下循环的条件

```c
    while((ch = buffer_get_next(buf)) != EOF)
```
print一下ch，显示255。什么，255？不应该是-1么，立马写了个测试程序测试下，发现EOF的值确实是-1，但赋值给char型的ch后就变成255了，然后再判等的时候永远是false。网上稍微查了下，发现确实如此，ARM下面的gcc默认char为unsigned char，不像x86下char默认为signed char，而C标准中规定char是有符号数还是无符号数可以由编译器自己决定。

至此问题已经清楚了，<del>图个方便我把EOF全部改成了255，再编译</del>，编译时加上`-fsigned-char`选项，程序运行正常，不会段错误了。

不得不说C语言真是博大精深，永远也学不透……