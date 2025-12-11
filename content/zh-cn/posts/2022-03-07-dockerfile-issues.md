---
title: "Dockerfile踩坑记"
date: 2022-03-07
slug: dockerfile-issues
tags:
- docker
---

最近在公司里写Dockerfile，需要build一个包含conda环境的image，踩了无数坑，只能感叹自己学艺不精，特此记录。

<!--more-->

## `COPY`指令的环境变量问题

来看以下Dockerfile：

```
FROM ubuntu:20.04

COPY test.txt $HOME/
```

`$HOME`默认为`/root`，直觉上会认为`test.txt`被拷贝到了`/root`下，事实上并不是，`test.txt`被拷贝到了`/`下。这是因为`COPY`指令只认在Dockerfile中出现过的`ENV`指令的环境变量，与一般Shell的环境无关，以下Dockerfile则没有问题：

```
FROM ubuntu:20.04

ENV HOME=/root
COPY test.txt $HOME/
```

## `ENTRYPOINT`指令的格式问题

`ENTRYPOINT`指令接收字符串或是JSON格式。由于是JSON格式，不能使用单引号，一定要使用双引号。最坑的是如果你用了单引号，Docker不会报错，只会把它当成字符串作为`SHELL`指令（默认`/bin/sh -c`）的参数，于是就会报类似如下错误：

```
[/entrypoint.sh]: No such file or directory
```

## `.bashrc`问题

这个问题其实和Docker关系不大。前文说过，我是在做与conda环境相关的image，主要参考了[这篇文章](https://pythonspeed.com/articles/activate-conda-dockerfile/)，但是死活不成功。大致上是，在使用conda前，需要跑一次`conda init bash`命令，这个命令会往`~/.bashrc`里写入一些conda的初始化命令，根据文章里的步骤，我调用了这个命令，使用`SHELL`指令切换到了`bash --login -c`，然而conda始终报错说shell没有初始化。

查阅一番资料后终于搞清楚了这个问题，主要是shell可以根据是否interactive，是否是login shell分成四种：

|类型|初始化脚本|范例|
|---|---|---|
|interactive login shell|`/etc/profile`，随后`~/.bash_profile`，`~/.bash_login`，`~/.profile`中最先找到的那个|通过SSH登录的bash，或`bash --login`|
|interactive non-login shell|`~/.bashrc`|登录后再调用一次bash|
|non-interactive login shell|`/etc/profile`，随后`~/.bash_profile`，`~/.bash_login`，`~/.profile`中最先找到的那个|`bash --login -c`|
|non-interactive non-login shell|无|`bash -c`或是`bash script.sh`|

在bash中可以通过`shopt login_shell`查看当前shell是否是login shell。

分的情况太多太混乱，于是为了统一interactive shell和non-interactive shell，一般会在`~/.bash_profile`或者`~/.profile`里加这么几行：

```bash
if [ -f "$HOME/.bashrc" ]; then
    . "$HOME/.bashrc"
fi
```

这样就能做到不管是不是interactive shell都会执行`~/.bashrc`了。

造成我这个问题的原因在于我在Dockerfile里将`HOME`变量设到了一个临时文件夹（因为Openshift运行容器的UID近乎不可控的随机值，无法读写/root），同时我又没复制`~/.bash_profile`，导致`conda init bash`虽然写入了`~/.bashrc`，但是在non-interactive login shell中根本不执行。

解决方法也很简单，拷贝一个`.bash_profile`过来就好了。

这个问题在后来写`entrypoint.sh`时也遇到了，一开始的`entrypoint.sh`长这样：

```
#!/usr/bin/env bash

exec "$@"
```

这样运行的是一个non-interactive non-login shell，不会运行任何初始化脚本，需要改成：

```
#!/usr/bin/env -S bash --login

exec "$@"
```

才能正常工作。

Shell基本功掌握不牢，真是惭愧，惭愧。