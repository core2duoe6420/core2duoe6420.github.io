---
title: "Settle Down"
date: 2014-09-09
slug: settle-down
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

Back when I first started playing with VPS, I set up a blog, but abandoned it after writing only a few posts. Blogs are more suitable for technical articles; if I post about my feelings, who would come back to read them? But every time I finish tinkering with something, I feel exhausted. It’s really “tinkering”: I always run into all sorts of strange problems, and then I don’t feel like writing anymore. So how long I can keep this blog going, I honestly have no idea. Of course, I was also influenced by wjx. That guy has been really into blogging recently, doing it with great flair, [link here](http://devopstarter.info), which got me wanting to do it as well.

<!--more-->

Take setting up this blog as an example: I spent an entire day on it. Now that Markdown is popular, writing blog posts is much more convenient than before. Today I happened to stumble on the [Cmd Markdown Editor](https://www.zybuluo.com), and I was blown away: Markdown actually has so many advanced features. Syntax highlighting is one thing, but it even has LaTeX math formula extensions—that’s the feature I’ve been dreaming of. Reading further, I found it even supports drawing flowcharts directly, which instantly made up my mind to use Markdown for blogging.

The key is to find a decent engine and blog theme. I don’t do frontend, so writing one myself is out of the question. If I pick something that’s too complicated to set up, it would be a nightmare to tinker with, and I’d be too lazy to bother. In the end I still spent a whole day. I tried several options, compared them, and finally chose the current solution. Here are my impressions of the ones I tried.

- [Ghost](https://ghost.org): This is the system wjx uses, with the ghostion theme. The backend is built with node.js, and supposedly the performance is great. I’m not particularly interested; there are only a few people who will ever visit this blog anyway, so performance is secondary—usability is king. What I dislike most about ghostion is the code style: it’s unbelievably ugly. The font is bold and huge, and the line spacing is large as well. Just a few lines of code can take up half the screen. I checked the CSS and it doesn’t look easy to tweak, so I scrapped it.

- [Strapdown.js](http://strapdownjs.com/): With this JS, you can directly convert markdown into HTML for display, and it will download the related theme and CSS online. The style is very similar to GitHub. I tried it and it does look quite impressive; in terms of style, Strapdown.js left the best impression on me. But its markdown engine is a bit weak. I tested it using the file on the Cmd Markdown homepage, and it doesn’t support endnotes. As for flowcharts, forget it. Aside from Cmd Markdown and a foreign online editor called [StackEdit](https://stackedit.io/), almost no engines support flowcharts. But those online editors can’t really be used as a blogging system. Also, Strapdown.js only produces static pages; to build a blog system, you’d still have to create your own index and such. I went back and forth between Strapdown.js and JustWriting for quite a while, and in the end I gave up on Strapdown.js.

- [Farbox](https://www.farbox.com/)+Dropbox: Dropbox is a fairly well-known cloud storage service. Farbox accesses markdown files stored in Dropbox via its API and then parses them. It’s said that this system is pretty good, but Farbox is paid: the cheapest plan is $1 per year, which is not expensive, but the monthly traffic is too small—only 100M. I might not even use that up, but it still makes me uncomfortable. The higher plans are more expensive, and I already have an idle VPS—why not use it? Also, Dropbox is blocked (it seems only the DNS is poisoned), so accessing it is inconvenient. So this solution was abandoned, and I never tested how good Farbox’s Markdown engine is.

- [JustWriting](https://github.com/hjue/JustWriting): I found JustWriting while searching for Farbox-like systems. Inspired by Farbox, the author also built a blog system that reads files directly from Dropbox. There are a few similar solutions; the other two looked more complicated, and JustWriting was easier to install, so I picked it. But the installation process was still frustrating. The author probably changed the directory structure but didn’t update the rewrite rules in the Apache config file, causing the highlight feature to never work; I wrestled with it for a long time. JustWriting offers two themes. One is the author’s own, which I found uncomfortable: the font is too small, and the code is tightly stuck to the border with no margin. After switching to the theme I’m using now, it immediately felt much cleaner. The default width was too narrow and the font a bit small, but after inspecting the elements I found it wasn’t hard to fix. A quick tweak of width and font-size, and I decided to stick with it. JustWriting’s markdown engine is not perfect either; it doesn’t even support the simplest strikethrough (i.e., <del>like this</del>, which is written with `~~` in markdown), though it does support endnotes. By comparison, this is still better, since strikethrough can be replaced with the HTML `<del></del>` tag, whereas endnotes are not that simple. Flowcharts are still unsupported, but considering I’ll almost never need them, that’s acceptable. This theme has another issue: it doesn’t have table styles. I copied some table styles from Cmd Markdown, and now it’s barely usable—though I’m not sure if there are other bugs.

The math formula feature is not related to the Markdown engine; it’s implemented by a JS library called MathJax. Here’s a [basic tutorial](http://meta.math.stackexchange.com/questions/5020/mathjax-basic-tutorial-and-quick-reference). To use MathJax, you just need to include this snippet on your site:

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

Let’s try a formula for practice, Gauss’s formula:

$$\iiint_\Omega\left(\frac{\partial P}{\partial x}+\frac{\partial Q}{\partial y}+\frac{\partial R}{\partial z}\right)\mathrm{d}V = \Huge \unicode{x222F}\normalsize_{\partial \Omega}\left(P\cos\alpha+Q\cos\beta+R\cos\gamma\right)\mathrm{d}S$$

MathJax does not yet support the double closed surface integral symbol, so we can only use a Unicode character instead. It looks a bit ugly and rather unharmonious…

Finally, a quick test of tables:

|title 1|title 2|title 3|
|:---|:---:|---:|
|row 1 column 1|row 1 column 2|row 1 column 3|
|row 2 column 1|row 2 column 2|row 2 column 3|
|row 3 column 1|row 3 column 2|row 3 column 3|

That’s it. Overall, the result is still pretty good. But this theme has one more drawback: it doesn’t support comments, though that doesn’t really matter.

Lastly, I’d also like to thank the author of JustWriting.