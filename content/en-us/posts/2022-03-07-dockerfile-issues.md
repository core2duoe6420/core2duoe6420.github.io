---
title: "Dockerfile Pitfalls"
date: 2022-03-07
slug: dockerfile-issues
tags:
- docker
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

Recently I’ve been writing Dockerfiles at work to build an image that contains a conda environment. I ran into countless pitfalls and can only lament my lack of skill; I’m recording them here.

<!--more-->

## Environment variable issues with the `COPY` instruction

Take the following Dockerfile:

```dockerfile
FROM ubuntu:20.04

COPY test.txt $HOME/
```

`$HOME` defaults to `/root`, so intuitively you’d think `test.txt` would be copied under `/root`, but in fact it’s not: `test.txt` is copied under `/`. This is because the `COPY` instruction only recognizes environment variables that have appeared in `ENV` instructions in the Dockerfile; it has nothing to do with the usual shell environment. The following Dockerfile, by contrast, works fine:

```dockerfile
FROM ubuntu:20.04

ENV HOME=/root
COPY test.txt $HOME/
```

## Format issues with the `ENTRYPOINT` instruction

The `ENTRYPOINT` instruction accepts either a string or JSON format. Since it’s JSON format, you cannot use single quotes; you must use double quotes. The most insidious part is that if you use single quotes, Docker will not report an error. It will simply treat them as a string argument to the `SHELL` instruction (by default `/bin/sh -c`), which then leads to an error like:

```text
[/entrypoint.sh]: No such file or directory
```

## The `.bashrc` issue

This problem is actually not very specific to Docker. As mentioned earlier, I was building an image related to a conda environment, mainly following [this article](https://pythonspeed.com/articles/activate-conda-dockerfile/), but I just could not get it to work.

Roughly speaking, before using conda you need to run `conda init bash` once. This command writes some conda initialization commands into `~/.bashrc`. Following the steps in the article, I invoked this command and used the `SHELL` instruction to switch to `bash --login -c`, yet conda kept complaining that the shell had not been initialized.

After looking up some documentation, I finally figured it out. Essentially, shells can be divided into four types depending on whether they are interactive and whether they are login shells:

|Type|Initialization scripts|Example|
|---|---|---|
|interactive login shell|`/etc/profile`, followed by the first found among `~/.bash_profile`, `~/.bash_login`, `~/.profile`|bash logged in via SSH, or `bash --login`|
|interactive non-login shell|`~/.bashrc`|run bash again after login|
|non-interactive login shell|`/etc/profile`, followed by the first found among `~/.bash_profile`, `~/.bash_login`, `~/.profile`|`bash --login -c`|
|non-interactive non-login shell|none|`bash -c` or `bash script.sh`|

In bash you can run `shopt login_shell` to see whether the current shell is a login shell.

There are too many cases and it’s too confusing, so in order to unify interactive shells and non-interactive shells, people usually add the following lines to `~/.bash_profile` or `~/.profile`:

```bash
if [ -f "$HOME/.bashrc" ]; then
    . "$HOME/.bashrc"
fi
```

This way `~/.bashrc` is executed regardless of whether the shell is interactive.

The cause of my problem was that in the Dockerfile I set the `HOME` variable to a temporary directory (because on OpenShift the UID that runs the container is an almost uncontrollable random value and cannot read/write `/root`), and at the same time I did not copy over `~/.bash_profile`. As a result, although `conda init bash` wrote to `~/.bashrc`, it was never executed in the non-interactive login shell.

The solution is simple: just copy a `.bash_profile` over.

I later ran into the same issue when writing `entrypoint.sh`. The initial `entrypoint.sh` looked like this:

```bash
#!/usr/bin/env bash

exec "$@"
```

This runs a non-interactive non-login shell that does not execute any initialization scripts, and needs to be changed to:

```bash
#!/usr/bin/env -S bash --login

exec "$@"
```

for it to work properly.

I really have to admit my basic shell skills are weak—very embarrassing indeed.