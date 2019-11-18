---
title: vscode才是真正的markdown神器
date: 2018-05-22 16:16:07
tags: [vscode, markdown]
categories: techniques
---
markdown作为程序员必备的文档编辑神器，突出体现了程序员“关注内容轻格式”的风格，只采用了简单的标记符号，就可以很方便地写出格式漂亮规范统一的文档，岂不快哉！！最开始写markdown的时候，尝试了很多markdown编辑器，甚至包括一些markdown的笔记，后来，当vscode真正全面支持markdown语法以及同步实时预览之后，我才发现了新大陆，可以说vscode基本算是秒杀其他所有markdown编辑器的存在了，下面一一道来。

### 同类markdown编辑器比较
- 为知笔记
    
    为知笔记算是市面上一款不错的markdown编辑器了，但是使用过后我也发现了几个比较明显的短板。首先它虽然声称是跨平台的，但是在linux上没有相应的rpm或者deb安装包，而是要到github主页上把项目clone下来自己编译才行，构建工具是cmake，所以相对要麻烦一些，对于一些拿markdown轻写作的非程序员可能有一点小小的门槛。其次，为知笔记的网络同步功能是要收费的，否则你只能用它当作一个本地的markdown编辑器来用。最重要的一点，就是在markdown编辑界面没有实时预览，要点击一个按钮才能出来对应的markdown选然后的文章，并且在编辑界面格式采用的竟然是反人类的microsoft doc的编辑方式，也就是说，你在markdown编辑的时候可以像写word一样设置各种样式和字体大小，虽然最后渲染成markdown文章后长得都一样。。。所以这个设计感觉有点蠢，没用多久就被我抛弃了。

- 作业部落

    这个也和为知笔记差不多，支持web编辑，但是导出html和pdf之类的要收费，后来也就没怎么关注。

- 有道云笔记

    这个是我用的比较多的，现在也还在用。有道云笔记是一款支持编写markdown笔记的软件，有网页版，也可以安装客户端，印象笔记就不支持markdown，所以就没用。但是有道云笔记分享出去的markdown笔记经常会无法访问，所以只适合自己记笔记用，不适合做分享。

### vscode写markdown有什么过人之处？

visual studio code写markdown的优点总结为以下几点：

1. 方便的在线预览

    虽说在线实时预览是markdown编辑器基本都有的功能，但是，作为一款市面上主流的coding editor，能够通过`Ctrl K + V`直接调出实时markdown预览，感觉是不是棒棒哒！毕竟原生支持markdown实时编辑预览的代码编辑器还是非常有限的。

    ![markdown预览](http://ww1.sinaimg.cn/large/a3543933gy1g926g1ms7ij21p40zzn7u.jpg)

2. 多样化的插件带来的功能扩展

    我们知道，vscode最大的优点就是开源带来的丰富的第三方插件，通过这些插件，我们可以大大丰富vscode的markdown编辑功能。随便一搜，markdown的插件就数不胜数。
    
    ![markdown插件](http://ww1.sinaimg.cn/large/a3543933gy1g926gs005jj207r0yz40k.jpg)

    - Markdown All in One

        这个是下载量最多的一款插件，功能比较齐全，支持github上所有的markdown格式，同时可以导出html等方便分享，同时还有很多方便的快捷键，并且对于github这种不支持`[toc]`语法的也提供了一键生成目录链接的功能，很方便。

    - Markdown PDF

        这个插件很强大，可以在vscode里一键导出html, pdf, PNG三种格式支持，配置起来也比较容易，直接搜索插件按照说明完成配置即可使用。

    - vscode-pandoc

        这个插件和上面的插件功能差不多，但是可以支持markdown导出成Microsoft Word文档，这个功能应该是很多的开发者比较期望的功能了。毕竟，你写的`.md`文件分享给其他人的时候人家对方可能会一脸懵逼地问你这个东西要怎么打开。。。安装这个插件首先需要在电脑里装pandoc，具体的安装参考插件里的说明页面即可。

3. 跨平台

    无论你用的是Mac OS, 还是各种Linux的发行版，还是Windows，你都可以安装Visual Studio Code。因为它是微软开源的，在各个平台都可以下载安装。这样，你就不需要担心在工作中因为操作系统的切换或者换了工作环境导致你不得不去更改以前已经习惯了的Markdown编辑器了。
