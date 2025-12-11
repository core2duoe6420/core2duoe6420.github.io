---
title: "安家落户"
date: 2014-09-09
slug: settle-down
---

以前刚玩VPS的时候做过一个blog，但是没写几篇文章就被我荒废了。博客还是比较适合发技术文章吧，要是发心情文章又有谁回来看，但我每次折腾完一个东西后都感觉很累，真的是折腾，总是遇到莫名其妙的问题，然后也就不高兴写文章了，所以这个博客能坚持多久，我也不知道。当然，也是受到了wjx的影响，这货最近写博客写的很起劲，搞得有声有色的，[链接在此](http://devopstarter.info)，所以弄得我也想搞了。

<!--more-->

就说弄这个博客吧，折腾了整整一天。因为现在Markdown兴起，写博客比原来要方便太多了，今天无意间看到了[Cmd Markdown编辑器](https://www.zybuluo.com)，惊为天人，原来Markdown还有这么多高级功能，语法着色这种也就算了，居然还有LaTeX的数学公式扩展，这可是我梦寐以求的功能，再往下看居然还有直接绘制流程图的功能，瞬间让我打定了用Markdown写博客的主意。

关键是要找个好一点的引擎和博客模板，我不是搞前端的，要我自己写根本不可能，要是找个搭起来太复杂的，要折腾死了，也不高兴搞。结果最后还是折腾了一天，找了几种方案，对比了一下，最终选择了现在这个方案。大致介绍一下我试过的几个方案的感受。

- [Ghost](https://ghost.org)：也就是wjx用的那套系统，配上ghostion主题，后台是用node.js，据说性能很好，我是不怎么感冒，这博客一共没几个人会来看的说，性能都是其次，好用才是王道。ghostion让我最不舒服的就是代码的样式，实在是丑到爆了，字体又粗又大，行间距也大，几行代码能占据半块屏幕，我看了下CSS，要改似乎也不容易，枪毙掉。
- [Strapdown.js](http://strapdownjs.com/)：用这个js可以直接把markdown转换为html显示，会联机下载相关的主题和CSS，风格和github很像，我试用了下确实很大气，论风格Strapdown.js给我的感觉最好，但是它的markdown引擎功能弱了点，我用Cmd Markdown首页的那个文件试了下功能，它不支持尾注，流程图神马的就不说了，除了Cmd Markdown和国外的一个在线编辑器[StackEdit](https://stackedit.io/)外，流程图基本没有哪个引擎支持的，但这些在线编辑器又不能用来当博客用。另外，Strapdown.js做出来的是静态网页，要做成博客系统还得自己做个目录啥的，我在Strapdown.js和JustWriting间考虑了很久最后还是放弃了Strapdown.js。
- [Farbox](https://www.farbox.com/)+Dropbox：Dropbox是个比较著名的网盘，Farbox通过Dropbox的API获得存放在Dropbox里的markdown文件然后解析，据说这套系统不错，但Farbox要收费，最便宜的$1一年，不贵，但每个月流量太少了，才100M，虽然也未必用的掉，但我总觉得心里不舒服，再上去的套餐就贵了，我还有VPS闲置着呢，干啥不用？而且Dropbox被墙了（好像只是DNS被污染了），访问也不方便。这个方案就被放弃了，Farbox的Markdown引擎做得如何也没有测试过。
- [JustWriting](https://github.com/hjue/JustWriting)：JustWriting是在搜Farbox那套系统时搜到的，作者受到启发也用Dropbox写了一套直接从Dropbox取文件的博客系统。类似的这样的方案还有几个，另外两个看上去比较复杂，JustWriting安装起来比较方便，所以就选它了。但实际上安装的过程也很折腾，作者可能是修改过目录层次但没把apache的配置文件里rewrite的规则改掉，导致highlight功能一直无效，折腾了好久。JustWriting提供了两个主题，一个是作者自己用的，那个主题我感觉不舒服，字体太小，代码会与边框紧紧贴在一起，没有margin，切换到我现在用的这个主题，顿时感觉清爽了不少，初始的主题太窄，字体也略小，不过查看了下元素发现改起来不难，简单修改了下width、font-size之后，就决定用这个了。JustWriting的markdown引擎也不完美，连最简单的删除线（就是<del>这种</del>，markdown用`~~`括住）都不支持，不过居然支持尾注。相比之下还是这个好点，毕竟删除线可以用`<del></del>`的HTML标签代替，尾注就没那么简单了。至于流程图这种，照样不支持，考虑到几乎不可能用到，也就算了。这个主题还有一个问题就是没有table的样式，我从Cmd Markdown那里抄了点table的样式过来，现在勉强能用了，就是不知道还有没有其他bug。

至于数学公式的功能，跟Markdown引擎没关系，是一个叫MathJax的js完成的，这是[初级教程](http://meta.math.stackexchange.com/questions/5020/mathjax-basic-tutorial-and-quick-reference)，要用MathJax只要在网上加入这段代码就可以了：

```html
<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default">
MathJax.Hub.Config({
    tex2jax: {
        inlineMath: [ ["$","$"] ]
    },
    extensions: ["jsMath2jax.js", "tex2jax.js"],
    messageStyle: "none"
});
</script>
```

来一个公式练练手，高斯公式：

$$\iiint_\Omega\left(\frac{\partial P}{\partial x}+\frac{\partial Q}{\partial y}+\frac{\partial R}{\partial z}\right)\mathrm{d}V = \Huge \unicode{x222F}\normalsize_{\partial \Omega}\left(P\cos\alpha+Q\cos\beta+R\cos\gamma\right)\mathrm{d}S$$

MathJax现在还不支持二重封闭曲面积分符号，所以只能用一个unicode字符代替，看上去丑了点，一点都不和谐……

最后试一下表格：

|title 1|title 2|title 3|
|:---|:---:|---:|
|row 1 column 1|row 1 column 2|row 1 column 3|
|row 2 column 1|row 2 column 2|row 2 column 3|
|row 3 column 1|row 3 column 2|row 3 column 3|

就是这样，总的来说效果还是不错的。不过现在这个主题还有个缺点，就是不支持评论了，不过这个无所谓了。

最后还要感谢JustWriting的作者，