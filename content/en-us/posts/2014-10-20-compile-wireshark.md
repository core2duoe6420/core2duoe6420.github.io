---
title:  "Compiling Wireshark on Windows"
date: 2014-10-20
slug: compile-wireshark
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

The project I was working on used the IP header Option field and defined a custom type. By default Wireshark shows this as Unknown. To make the demo look better, I added some code to let Wireshark support custom Option types. Since this is not a standalone protocol, it cannot be extended with Lua and similar mechanisms, so Wireshark has to be recompiled. Compilation on Linux is very simple, but on Windows it is much more cumbersome, hence this article.

<!--more-->

## References
The main reference is the section [`Win32/64: Step-by-Step Guide`](https://www.wireshark.org/docs/wsdg_html_chunked/ChSetupWin32.html) in the Wireshark documentation. Of course, I did not follow it to the letter; otherwise it felt even more tedious.
Some blog posts on the web are also useful, such as [this one](http://www.360doc.com/content/14/0703/17/11764545_391768149.shtml).

## Environment

For compiling this type of software on Windows, it’s better to use a clean environment to ensure success in one go, otherwise when something goes wrong you won’t even know where the problem is. I did a fresh install of Windows 7 x64 SP1 in a virtual machine, then installed Visual Studio 2010 Ultimate, and then updated to Visual Studio 2010 SP1 (installing SP1 is even slower than installing VS itself). Here I did not install the Express edition as suggested in the Wireshark docs (that one is free, the one above costs money; let’s not get into that topic…), nor did I install the Windows SDK (I felt this part in the documentation was extremely troublesome due to Microsoft’s own BUGs, plus what you download from Microsoft’s site is a web installer, and the lab’s network is horribly slow to Microsoft servers; in addition, I figured VS should already include the Windows SDK, so I skipped it, and indeed no issues came up later).

After that I installed the 32‑bit version of Python 2.7.8. Then I installed QT 5.3; I used the [`qt-opensource-windows-x86-msvc2010_opengl-5.3.2`](http://mirror.bjtu.edu.cn/qt/official_releases/qt/5.3/5.3.2/qt-opensource-windows-x86-msvc2010_opengl-5.3.2.exe) package downloaded from Beijing Jiaotong University’s mirror (mainly because the download was fast).

Finally I installed Cygwin. According to the documentation, certain components must be installed when setting up Cygwin. Below are the components I installed:

- `Archive/unzip`
- `Devel/bison`
- `Devel/flex`
- `Devel/git`
- `Interpreters/perl`
- `Utils/patch`
- `Web/wget`
- `Interpreters/m4 `

In short, I installed everything that appeared in the docs, whether required or recommended. Here I ran into a ridiculous issue: after installing according to this list, compilation still failed with an error saying it could not find the `u2d` command, which should be similar in function to `unix2dos`. I then searched for `unix2dos` in the Cygwin package list but found nothing. Some posts online said this command should be in the `cygutils` package, but that package was already installed. I repeatedly uninstalled and reinstalled it, changed mirrors, but nothing worked. Eventually I discovered the command was actually in the `dos2unix` package… so the `dos2unix` package must also be installed!

All installation paths for the tools above were left at their defaults; do not change any of them. This avoids a lot of trouble.

## Compilation
Once the environment is ready, we can start compiling.

Step 1: Download the Wireshark source code. I used version 1.12.1, extracted it, and put it under `C:\wireshark` (any path is fine). Then edit the `config.nmake` file in the source tree root directory, mainly modifying the following parameters:

- `VERSION_EXTRA`: custom description that will be appended to the version number after compilation.
- `QT5_BASE_DIR:`: Qt5 installation directory. This directory must contain `bin\qmake.exe`. If you used the Qt 5 installer mentioned above and installed it to the default location, it should be `C:\Qt\Qt5.3.2\5.3\msvc2010_opengl`.

I did not touch other parameters, such as `MSVC_VARIANT` or `WIRESHARK_TARGET_PLATFORM` as described in the docs. If all installation paths are default, `config.nmake` will automatically set all parameters.

Step 2: Run the VS script to set the build target to x86, i.e., 32‑bit. A 32‑bit Wireshark build can run on both 32‑bit and 64‑bit Windows, whereas a 64‑bit Wireshark build can only run on 64‑bit Windows. Method: open cmd and run this file:

    C:\Program Files (x86)\Microsoft Visual Studio 10.0\VC\vsvars32.bat
    
Step 3: Check for and download the required components. From the VS group in the Start menu, open `Visual Studio 命令提示(2010)` (if you open a regular cmd, `nmake` will not be found unless you add it to the `PATH` environment variable), then go to `C:\wireshark` (the Wireshark source directory) and run:

    nmake -f Makefile.nmake verify_tools
    
The first run will start with an error about missing `current_tag.txt` and so on; this does not matter. As long as programs like `bison` and `perl` are OK below, you are fine. Then run:

    nmake -f Makefile.nmake setup
    
This command installs all dependency libraries, including the assembler, DNS library, GeoIP, GTK, etc. These packages are all downloaded from the Internet (hence the need for `wget` earlier). If the download is unbearably slow, you can switch to a different network environment, download them manually, and then place them in the `C:\Wireshark-win32-libs-1.12` folder (this is the default; the directory can be changed in `config.nmake`).

After all dependencies are installed, run the check again; there should be no errors.

Step 4: Compile. Run:

    nmake -f Makefile.nmake distclean
    nmake -f Makefile.nmake all
    
If everything goes smoothly, the build should succeed. After compilation, two folders will be created under the Wireshark source root: `wireshark-gtk2` and `wireshark-qt-release`. The usual Wireshark program is under `wireshark-gtk2`; you can run `Wireshark.exe` directly (of course, `WinPcap` must be installed on the system; otherwise it’s just a shell). The `wireshark-qt-release` folder contains the Qt‑based Wireshark build, whose executable is `qtshark`, a preview version of Wireshark 2.

## Generating the installer

To run the compiled Wireshark on other computers, you need to generate an installer. First install `NSIS`; it’s best to download the stable 2.46 version, as the 3.0 beta has BUGs. The 32‑bit version is sufficient. During installation, it’s best to use the default location as well, so you don’t have to modify `config.nmake` again. Then download `vcredist_x86.exe` from Microsoft’s website and put it into the `C:\Wireshark-win32-libs-1.12` folder. Then run:

    nmake -f Makefile.nmake packaging
    
After it succeeds, go to the `C:\wireshark\packaging\nsis\` folder to find the generated installer. This installer has `WinPcap` bundled already and is basically the same as the Wireshark installer you download from the Internet.

At this point, all work is complete.