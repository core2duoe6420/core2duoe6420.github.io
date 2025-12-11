---
title: "Cross-Compiling an ARM Native GCC"
date: 2014-09-10
slug: cross-compile-arm-natvie-gcc
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

Some time ago I learned to use IP tunnel to build a tunnel so that machines without native IPv6 could use IPv6. I was itching to get IPv6 at home in advance as well, so I finally bit the bullet and bought the R6300v2 I’d been eyeing for a long time. With ddwrt on it, IPv6 worked smoothly.

<!--more-->

The R6300v2 is equipped with BCM4708, an ARM Cortex-A9 dual-core 800MHz CPU. The latest ddwrt kernel version is 3.10.25, which is quite new, so I wondered if I could run GCC on the router and compile programs there. I searched online for a long time but couldn’t find any GCC that could run directly on the router. However, there were plenty of cross-compilers that could generate binaries runnable on the router; openwrt has tons of them.

I had no real cross-compilation experience, but during winter vacation I read a book called “深度探索Linux操作系统 系统构建和原理解析” (hereafter referred to as “简析”). Back then I followed the book step by step and did manage to succeed, but I didn’t understand much of what was going on. So I had the idea to build from scratch a GCC that could run on the router, which started a four-day-long hacking journey.

## Environment

Host machine: `Ubuntu 14.04 64-bit, gcc 4.8.2`

Toolchain versions: `binutils-2.24 gcc-4.9.1 glibc-2.19 linux-3.10.25` (because the ddwrt kernel I installed is 3.10.25)

Compiling gcc requires three extra libraries: gmp, mpc, mpfr. After downloading and extracting them, rename the directories from gmp-version, mpc-version, mpfr-version to gmp, mpc, mpfr, and then put them under the gcc-4.9.1 directory. Newer gcc versions can correctly handle the locations of these libraries without extra options (gcc-4.7.4 needs `--with-mpfr-include` and `--with-mpfr-lib` during configure; that was one of the lessons learned while hacking on this).

The first thing in cross-compilation is to determine the target machine’s triplet, i.e., arch-vendor-os. On Ubuntu this triplet is `x86_64-pc-linux-gnu`, while on the R6300v2, after some experimentation, I ultimately used `arm-ddwrt-linux-gnueabi`. This triplet is very important because it affects how the C library is built. You can often see `arm-none-eabi` online, which means there is no operating system and newlib is used as the C library. I also tried this triplet during my experiments, and later realized that without an operating system this clearly wasn’t what I needed. Then there is the trailing `gnueabi`, and also the variant `gnueabihf`, which indicates that the CPU has an FPU; I guess hf stands for hard float. But supposedly routers don’t need floating point operations, so the R6300v2 has its FPU disabled -:(… Another type is the cross-compiler I found online: `arm-openwrt-linux-uclibcgnueabi`, which means it uses uclibc as the C library. ddwrt also comes with uclibc, so binaries compiled with this toolchain can run on the R6300v2. You can also see that the vendor field doesn’t really matter; you can name it however you like…

After extracting all source code, we need to apply a patch to gcc. My situation is somewhat special: the / mount point on the router is read-only, which means I can’t write files under conventional directories like /lib and /usr. Besides, the router’s internal storage can’t hold giants like gcc and glibc anyway. Fortunately, the R6300v2 has a USB port so I can plug in a USB storage device. I mount the USB drive under `/tmp/mnt/usb` (because /tmp is a ramfs and writable), put glibc’s library files under `/tmp/mnt/usb/lib`, and files under /usr into `/tmp/mnt/usb/usr`. However, the dynamic linker loaded by the OS when a program starts is hard-coded into gcc’s code, so we need to modify the dynamic linker’s location.

You can inspect the dynamic linker via `readelf -l a.out`. The output will contain a line like:

```no-highlight
[Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

This is the output on my Ubuntu x86_64 system, while what I want is:

```no-highlight
[Requesting program interpreter: /tmp/mnt/usb/lib/ld-linux.so.3]
```

Run the following code under the gcc-4.9.1 directory to apply the patch. The code comes from the LFS tutorial; I added linux-eabi.h because my target is ARM while LFS targets x86.

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

## Environment Variables

These are the environment variables I used in the commands; their origins have all been explained above. BUILD_DIR is the working directory for building. Note that the CROSS_TOOL directory contains cross-compilation tools that run on x86, while the SYSROOT directory contains programs that run on the R6300v2, including the final gcc, glibc libraries, and header files. The sysroot mechanism during cross-toolchain building is quite complicated, and I didn’t fully understand it either… As for why `/vita`, it’s because at first I followed “简析”, and the book uses `/vita`.

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

The first step in cross-compilation is always binutils. The goal is to build ld, as, etc., that run on x86 and generate ARM object code. Script:

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

Option explanations (all option explanations are based on my own understanding, and may not be correct—in fact, the chances of errors are quite high; I copied many from the internet but they worked):

- `--disable-nls`: nls means Native Language Support, i.e., only English messages during compilation.
- `--disable-libssp`: ssp seems to be some kind of stack protection; I don’t know why it’s disabled, but everyone disables it anyway…

## gcc round 1

The first gcc build aims to produce a minimal usable gcc without C library support, and then use it to build glibc. The key options are `--without-headers` and `--with-newlib`; they tell gcc to use its own built-in headers. Script:

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

Option explanations:

- `--target`: when the target and host parameters differ, cross-compilation is enabled. If host is not specified, gcc automatically uses the current triplet.
- `--prefix`: this only affects the installation location of the final gcc.
- `--disable-xxx`: in the first build we disable most features because we can’t build them without a C library.
- `--with-float=soft`: most `--with-xxx` options in gcc’s configure are used to specify default options that the resulting gcc will use when compiling other programs, such as `--with-cpu`, `--with-arch`, etc. With this option, when using the resulting gcc to compile other programs, it will automatically pass `-msoft-float` (not sure if this understanding is correct; you can refer to [Installing GCC: Configuration](https://gcc.gnu.org/install/configure.html) and [ARM Options](https://gcc.gnu.org/onlinedocs/gcc-4.5.3/gcc/ARM-Options.html)).
- `--with-local-prefix`: this adds the directory into gcc’s default header search path. The default is `/usr/local`. You don’t need to append `include` in the option, but in practice it looks in `/usr/local/include`. The reason for changing it here is the issue mentioned in the previous section.
- `--with-native-system-header-dir`: this option is actually not used in the first gcc build; it’s used starting from the second build. Together with `--with-sysroot` it specifies where to look for headers **during the build** and where the resulting gcc will look for system headers. So the final header search path is `$SYSROOT/$HOST_ROOT/usr/include`. For this we need to install both the Linux kernel headers and glibc headers to that location.

## Kernel Headers

Installing kernel headers is fairly straightforward. Pay attention to the ARCH parameter and the installation location; the reasons were explained when discussing gcc options above.

```no-highlight
cd ../linux-3.0.25
make mrproper
make ARCH=arm INSTALL_HDR_PATH=$SYSROOT/$HOST_ROOT/usr headers_install
```

## glibc

With the initial gcc in place, we can use it to build glibc. In glibc’s configure, `--host` indicates that we are building glibc to run on the target machine, so the resulting glibc cannot run directly on x86 Ubuntu. glibc has no concept of target.

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

Option explanations:

- `--prefix`: specifies the installation directory. glibc’s install scripts also embed this value into the libraries’ dependencies so that, on the target system, libraries will look for dynamic libraries in the corresponding directory. That’s why we don’t include $SYSROOT here, but instead specify the installation directory as $SYSROOT during `make install`.
- `--with-headers`: specifies the Linux kernel headers directory, i.e., the location where we just installed them.
- `libc_cv_*`: these three are used to “cheat” glibc. For details, see [Linux From Scratch 5.7 Glibc-2.19](http://www.linuxfromscratch.org/lfs/view/development/chapter05/glibc.html).

After glibc is installed, by default it puts headers under `$SYSROOT/$HOST_ROOT/include`. According to what was said above, we need to move them to `$SYSROOT/$HOST_ROOT/usr/include`:

```no-highlight
cp -r $SYSROOT/$HOST_ROOT/include $SYSROOT/$HOST_ROOT/usr/
```

## gcc round 2

Now that we have glibc, we can rebuild gcc and enable most features.

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

Option explanations:

- `--disable-decimal-float`: at one point the build complained that the target system does not support decimal-float, so I disabled it.
- `--disable-multilib --disable-libgomp`: without these, the build fails (libiberty and zlib errors; I don’t know the exact reason).
- `--with-native-system-header-dir`: this option now starts to take effect.

After this round, gcc as a cross-compilation tool is complete. Binaries compiled with it should run correctly on the R6300v2 (assuming glibc libraries are in the correct location, i.e., under `/tmp/mnt/usb/lib`). Write a hello world and inspect the ELF header via readelf:

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

The second binutils build aims to produce ld, as, etc. that run on the R6300v2. The key option is `--host`. Once this option is specified, the makefile will call `$TARGET-gcc` (i.e., `arm-ddwrt-linux-gnueabi-gcc`) and related cross tools to build binutils instead of using Ubuntu’s native compiler.

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

Option explanations:

- `--prefix`: since we want binutils that run on the R6300v2, we install them under the target directory.
- `--disable-werror`: without this, the build stops because warnings are treated as errors.

## gcc round 3

The third gcc build is, of course, to produce a gcc that runs on the R6300v2. Before starting, we need to copy files under `$SYSROOT/$HOST_ROOT/usr/include` to `$HOST_ROOT/usr/include` on Ubuntu, i.e., to `/tmp/mnt/usb/usr/include`, because the resulting gcc is to run on R6300v2, and `--with-sysroot` can no longer be set to `$SYSROOT` as in the previous two rounds; it must be `/`. So headers must be placed in the directory specified directly by `--with-native-system-header-dir`:

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

Option explanations:

- `--with-sysroot`: because this gcc is meant to run on the R6300v2, we can’t set sysroot to `$SYSROOT` as before; otherwise gcc would look for headers and libraries in the wrong locations on the router.
- `--disable-bootstrap`: gcc has a mechanism where it uses the newly built gcc to rebuild itself to see whether it can be built successfully. Enabling this here causes build errors for reasons unknown; again, refer to [Installing GCC: Configuration](https://gcc.gnu.org/install/configure.html).

## Conclusion

At this point gcc runs successfully on the R6300v2, though it’s not perfect yet: programs compiled with g++ trigger a segmentation fault on exit. I don’t yet know why, and I don’t feel like digging deeper for now. After all, gcc itself works fine, which fulfills my original goal. But turning the router into a fully functional build environment is still far off: it lacks make, various libraries, etc., and it’s not realistic anyway given the limited CPU power. It’s better to use the cross-toolchain to build everything and then deploy the binaries to the router.

During these four days of hacking, I compiled gcc about twenty times. The Pentium G2020 at home is too weak; a single build takes more than half an hour. It was exhausting both mentally and physically, and I was hanging on purely out of stubbornness. It also made me deeply realize that blind tinkering without theory is a waste of time. So I’d better study properly.

## Postscript

After finishing this article yesterday, I was excited to use the freshly built native gcc to compile some of my old programs as a test. If it couldn’t even handle my small programs, that would be a joke. I picked one of my earlier programs that doesn’t depend on other libraries—the PL0 interpreter I wrote in my third year—and started compiling. The first step was to build make for the R6300v2.

#### make

The make version I downloaded is 3.8.2; the latest seems to be 4.0.0. Building make is straightforward. Just install it under the target system’s directory.

```no-highlight
./configure --host=$TARGET --prefix=$SYSROOT/$HOST_ROOT/usr/
make -j $THREADS
make install
```

After make was built, I copied the PL0 code to the router and started building. Compilation succeeded! I was very happy. Then I ran it: `segmentation fault`! I was puzzled—this program runs fine on x86, so maybe gcc or something else was broken again. I added a few printf statements to quickly locate the problem and found it was in the lexical analysis part. The code was so old that I no longer remembered the internal structure, let alone fixing it. In a fit of determination, I decided to compile gdb for debugging, and hacked for a while again.

#### gdb

gdb depends on termcap, so first I downloaded `termcap-1.3.1.tar.gz` from the internet. This is a very old package. Build and install it:

```no-highlight
./configure --prefix=/vita/termcap --host=$TARGET
make
make install
```

termcap builds a static library `libtermcap.a`, so I got lazy and didn’t install it into the target system directory, but just dropped it into some directory.

With termcap built, I could then build gdb. First specify the termcap location with a few environment variables:

```no-highlight
export LDFLAGS="-static -L/vita/termcap/lib"
export CPPFLAGS="-I/vita/termcap/include"
export CFLAGS="-I/vita/termcap/include"
```

Then compile as usual:

```no-highlight
../gdb-7.8/configure --host=$TARGET --prefix=$SYSROOT/$HOST_ROOT/usr/
make -j $THREADS
make install
```

### The Problem

With everything ready, I started debugging with gdb and found the problem lay in a while loop; it appeared to be an infinite loop leading to out-of-bounds access and a segmentation fault. But this code runs perfectly on x86, so it shouldn’t be an algorithm issue. Since the loop never exited, I checked the loop condition:

```c
    while((ch = buffer_get_next(buf)) != EOF)
```

I printed ch and saw 255. What, 255? It should be -1. I immediately wrote a small test program. It turned out that EOF is indeed -1, but once assigned to a char variable ch, it became 255, so the equality test was always false. A quick search online showed that this is indeed the case: on ARM, gcc uses unsigned char by default for char, unlike x86 where char is signed by default. The C standard allows compilers to decide whether char is signed or unsigned.

At this point the problem was clear. <del>To be lazy I replaced all EOFs with 255 and recompiled</del>. Passing `-fsigned-char` at compile time also works. With that, the program ran fine and no longer segfaulted.

C is truly deep and profound; you can never completely master it…