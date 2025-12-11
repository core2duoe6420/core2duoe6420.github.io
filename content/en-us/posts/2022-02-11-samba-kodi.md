---
title: "Kodi Playback Stutter Caused by Samba Configuration"
date: 2022-02-11
slug: samba-kodi
tags:
- Kodi
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

Recently, when watching *The Good Doctor* with Kodi on my TV, the stuttering was so severe that it was basically unwatchable. At first I thought it was an issue with the source itself, maybe some highly compressed encoding parameters were used. I was quite puzzled though, because I had previously played HEVC 4K HDR videos without any issues, but I didn’t think too much of it. Later I found that almost all videos could no longer play normally, which clearly meant something had gone wrong somewhere, so I started another round of troubleshooting.

<!--more-->

First I thought through the changes I had made recently:

1. I changed the samba image from `dperson/samba` to `ghcr.io/jtagcat/samba`, because the former had not been updated for a long time and was missing some samba bug fixes and security patches, while the latter is a maintained fork of the former.
2. Kodi was upgraded to version 19.3.

Since I had upgraded Kodi, I went through the settings to see if there were any new options. I didn’t find anything noteworthy. Based on some posts online, setting the minimum SMB protocol to SMBv2 didn’t help either.

I saw people online saying Kodi performs much better with WebDAV than SMB, so I decided to switch to WebDAV first to confirm whether the Wi‑Fi connection itself had enough throughput for playback. I found a way to start an HTTP WebDAV server, configured it in Kodi, and found that playback worked fine. That confirmed the problem indeed lay in the SMB protocol.

To verify whether Kodi’s SMB client itself was at fault, I enabled a shared folder on Windows, put the video file in it, and played it via Kodi—no problem there either. So the issue was finally narrowed down to the samba server used on the NAS.

I remembered that when I first set up my NAS, Kodi could play videos normally, and at that time I was still using the `dperson/samba` image. So I checked the Dockerfiles of the old and new images to see if there were any differences:

```bash
diff \
    <(curl -s https://raw.githubusercontent.com/dperson/samba/master/Dockerfile) \
    <(curl -s https://raw.githubusercontent.com/jtagcat/samba/main/samba/Dockerfile)
```

Sure enough, I found something suspicious:

```diff
42a42
>     echo '   smb encrypt = desired' >>$file && \
55,56c55
```

The new version added an `smb encrypt = desired` configuration. After consulting the [official samba documentation](https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html#SERVERSMBENCRYPT), I learned that the difference between the default and desired options is:

- Leaving it as default, explicitly setting default, or setting it to if_required globally will enable negotiation of encryption but will not turn on data encryption globally or per share.
- Setting it to desired globally will enable negotiation and will turn on data encryption on sessions and share connections for those clients that support it.

It was highly likely that this option caused Kodi’s stuttering issue, so I added an environment variable in the container configuration: `GLOBAL4: 'smb encrypt = default'` (this is one of the configuration methods supported by the image) to revert this setting. After rebuilding the container and opening the video again in Kodi, playback was smooth—problem solved!