---
title: "网站已启用HTTPS和SPDY"
date: 2015-03-24
slug: equipped-with-https-and-spdy
---

使用HTTPS应该是大势所趋了，今后所有的网站都应该采用HTTPS加密连接来保证安全性和隐私。

Google的所有站点都已经全面启用了HTTPS，根据Google自己的说法，采用HTTPS带来的开销非常小。不过实际使用中，HTTPS的握手过程带来的延迟还是能明显感受到的，尤其是使用HTTP/1.1时通常要建立数条与服务器的连接，每条都要经过HTTPS握手过程。

<!--more-->

Google开发的SPDY协议（已成为HTTP/2.0的模板）可以很好的克服HTTP/1.1的许多不足。SPDY协议本身可以不使用HTTPS，但是Google的实现（包括apache的mod_spdy和Chrome浏览器）都强制要求SPDY基于HTTPS进行传输，因为SPDY是Google牵的头，大家基本也默认了这一点。SPDY在与服务器通讯过程中只使用一条连接，因此HTTPS的握手过程也只有一次，可以说基本上克服了使用了HTTPS带来的开销。SPDY使用的分帧、首部压缩等技术可以很好的克服HTTP/1.1时代的队首阻塞、首部过大等问题，使HTTP性能得到提升。

目前大多数最新的浏览器都支持SPDY协议，包括IE11、Chrome、Firefox。服务器方面，apache有mod_spdy，nginx的较新版本内置了对SPDY的支持，只需编译时加上`--with-http_spdy_module`选项即可。

我使用的版本是nginx 1.62，Ubuntu 14.04的更新源中的nginx版本较低，无法通过简单的配置启用SPDY，所以我加入了nginx的ppa源：`add-apt-repository ppa:nginx/stable`，这样再安装nginx就是1.62版本了，配置nginx，将`listen 443 ssl`修改为`listen 443 ssl spdy`即可，非常简单。

使用Chrome可以查看SPDY会话，方法是浏览器中输入`chrome://net-internals#spdy`。当存在SPDY会话时，会如下图所示：

![Chrome SPDY](https://res.cloudinary.com/core2duoe6420/image/upload/v1643913767/posts/spdy_ol6jej.png)

实际使用下来的感受还是不错的，相比原来使用HTTP时应该相差不多，海外服务器总会或多或少地丢包，TCP重传机制限制了SPDY的速度，不过还是在可接受的范围内。

使用HTTPS之后出现的另外一个问题就是页面上会存在请求其他域的HTTP连接，像MathJax、多说评论这些都是外链的，一旦存在这种HTTP连接，浏览器就会告警，Chrome上的绿色小锁上也会出现一个黄色惊叹号，让人很不舒服。我排查了之后发现，MathJax的CDN支持HTTPS，多说评论的主框架也支持HTTPS，但多说的头像和表情总是走HTTP，稍微了搜索了一下没有找到很好的解决方案，我又有强迫症，看不得这些警告信息，加上博客流量几乎为0，没人评论，干脆就把多说给禁了。

总感觉用别人的模板做的博客怪怪的，自己没法完全控制，想改些东西也不方便，所以等我什么时候下定决心了还是得自己写个，感觉工作量不小。

再配置nginx的过程中还看到了一些有意思的东西，使用nginx对Google进行反向代理，感觉比较实用，又一轮新的折腾要开始了。