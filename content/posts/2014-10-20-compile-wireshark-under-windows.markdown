---
title:  "Windows下编译Wireshark"
date: 2014-10-20
slug: compile-wireshark
---

手头的项目用到了IP首部的Option字段，而且自定义了一个type，正常情况下Wireshark会显示为Unknown，为了演示的时候效果更好一些，我添加了一些代码让Wireshark支持自定义的Option类型，因为这不是单独的协议，不能用Lua等扩展，所以必须重新编译Wireshark，Linux下的编译非常简单，Windows下要繁琐的多，所以就有了这篇文章。

<!--more-->

## 参考资料
最主要的参考资料是Wireshark文档中[`Win32/64: Step-by-Step Guide`](https://www.wireshark.org/docs/wsdg_html_chunked/ChSetupWin32.html)一节，当然我没有按照它上面说的完全照做，不然感觉更烦。
网上的一些博客文章也有参考价值，例如[这篇](http://www.360doc.com/content/14/0703/17/11764545_391768149.shtml)。

## 环境

在Windows下编译这种软件还是找一个干净的环境比较好，确保一部成功，不然错了都不知道哪里有问题。我在虚拟机中全新安装了Windows 7 x64 SP1，然后安装了Visual Studio 2010 Ultimate，之后更新了Visual Studio 2010 SP1（装SP1比装VS本身还慢）。这里我没有像Wireshark文档中说的装Express版本（这个是免费的，上面那个是要钱的，咱就不讨论这个问题了……），也没有安装Windows SDK（这个是我感觉文档中描述的非常麻烦的一部分，因为微软本身的BUG，再加上从微软网站上下载的是网络安装程序，实验室网络连微软服务器速度慢得一塌糊涂，再加上感觉装了VS应该附带了Windows SDK，所以跳过了，后来也确实没出什么问题）。

再之后安装Python 2.7.8的32位版本。然后安装QT5.3，我安装的是从北京交通大学镜像上下载的[`qt-opensource-windows-x86-msvc2010_opengl-5.3.2`](http://mirror.bjtu.edu.cn/qt/official_releases/qt/5.3/5.3.2/qt-opensource-windows-x86-msvc2010_opengl-5.3.2.exe)版本（主要是速度快）。

最后安装Cygwin，根据文档上写的，安装Cygwin的时候必须安装某些组件，以下是我安装的组件：

- `Archive/unzip`
- `Devel/bison`
- `Devel/flex`
- `Devel/git`
- `Interpreters/perl`
- `Utils/patch`
- `Web/wget`
- `Interpreters/m4 `

反正就是文档里有的，不管是必须装的还是推荐的都给装上。这里还遇到了个坑爹问题，按照这个列表装完之后编译时还是报错，说是找不到`u2d`这个命令，这个命令应该是跟`unix2dos`功能类似的命令，然后我就在Cygwin的列表里搜`unix2dos`，搜不到，网上有的说这个命令应该在`cygutils`包里，但是这个包已经装了，反复卸掉重装、换源装都不行，最后才发现原来在`dos2unix`这个包里……所以`dos2unix`这个包应该也是要装的！

以上所有的编译工具安装目录全部按照默认的来，一个都不要动，这样可以减少很多麻烦。

## 编译
搞定环境后开始编译。

第一步，下载Wireshark的源代码，我用的是1.12.1，解压后放到`C:\wireshark`文件夹下（其实随意）。然后编辑源代码根目录下的`config.nmake`文件，主要修改以下几个参数：

- `VERSION_EXTRA`：自定义的说明，编译完成后会跟在版本号后面。
- `QT5_BASE_DIR:`：QT5的安装目录，要求在这个文件夹下能找到`bin\qmake.exe`文件，如果用的是我上面的那个QT5安装程序并且安装位置默认，那么应该是`C:\Qt\Qt5.3.2\5.3\msvc2010_opengl`。

其它的参数都没动，像文档里说的`MSVC_VARIANT` `WIRESHARK_TARGET_PLATFORM`这种都没动，如果安装路径都是默认的，`config.nmake`会自动设置所有参数。

第二步，运行VS的脚本配置编译目标为x86，也就是32位的。32位的Wireshark编译出来32位64位Windows都能用，编译64位的Wireshark就只有64位的都能用了。方法：打开cmd，执行这个文件：

    C:\Program Files (x86)\Microsoft Visual Studio 10.0\VC\vsvars32.bat
    
第三步，检查并下载相应组件。从开始菜单的VS目录中打开`Visual Studio 命令提示(2010)`（直接打开cmd会找不到`nmake`，除非你设置了`PATH`环境变量），定位到`C:\wireshark`（Wireshark源代码所在目录），执行：

    nmake -f Makefile.nmake verify_tools
    
第一遍执行开始会报个错，说找不到`current_tag.txt`云云，这个没有关系，只要下面的`bison` `perl`这些程序都正常就ok。然后运行：

    nmake -f Makefile.nmake setup
    
这个命令会安装所有的依赖库，包括汇编器、DNS库、GeoIP、GTK等等，当然这些包都是从网上下载的（所以前面才需要装`wget`），如果下载速度慢到受不了，可以换个网络环境手工下载，然后放到`C:\Wireshark-win32-libs-1.12`文件夹下（默认配置，这个目录可以在`config.nmake`里改）。

所有依赖库安装完成后，再执行一边检查，应该不会报错了。

第四步，编译。执行

    nmake -f Makefile.nmake distclean
    nmake -f Makefile.nmake all
    
顺利的话就应该成功编译了。编译后会在Wireshark源代码根目录下的生成两个文件夹，`wireshark-gtk2`和`wireshark-qt-release`，其中`wireshark-gtk2`下就是通常的Wireshark程序了，可以直接运行`Wireshark.exe`(当然系统得安装了`WinPcap`，不然只是个壳子），`wireshark-qt-release`下放的是用Qt编译的Wireshark，执行文件为`qtshark`，是Wireshark2的预览版。

## 生成安装程序

为了能在别的电脑上运行编译出的Wireshark，需要生成一个安装程序。先安装`NSIS`，最好下载2.46稳定版，3.0测试版会有BUG，下载32位版即可。安装时最好也安装到默认位置，这样可以不用再去修改`config.nmake`文件。然后到微软网站上下载`vcredist_x86.exe`文件，放到`C:\Wireshark-win32-libs-1.12`文件夹下。然后执行：

    nmake -f Makefile.nmake packaging
    
成功后到`C:\wireshark\packaging\nsis\`文件夹下找生成的安装程序文件。这个安装程序已经自动打包了`WinPcap`，跟网上下载的Wireshark安装程序没什么两样。

至此所有工作完成。