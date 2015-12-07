---
layout: post
title: LyX 中文化指南(XeCJK+XeTeX, 支持中文编号)
date: 2011-09-09 23:02:49
categories: tech
tags: LaTeX,LyX,texlive,XeTeX 
description: 

---

在我的[上一篇文章](http://cherrot.com/2011/09/enable-full-chinese-after-installing-lyx-on-ubuntu/ "Ubuntu安装LyX后要做的那点事儿")里介绍了如何使LyX正确导出中文的PDF文件。不过我发现这样做对中文的支持太有限，不能使用其他字体，也不能使用系统的字体等等。还好我是个相当喜欢折腾的人（小胡有言曰“不折腾”，看来我有点反胡扯主义的味道），继续在Google上爬，这才知道原来使用XeTeX处理CJK才是王道，配合XeTeX自带的xeCJK宏包效果那叫一个赞。因为XeTeX最初是为Mac OS X设计的，原生支持Unicode。下面我将以图文形式详细说明如何设置LyX完美支持中文： _注：笔者使用系统平台为Ubuntu 11.04/Ubuntu11.10，LyX版本2.0.0，TexLive版本2009-11。本文以Ubuntu11.04/Ubuntu11.10的设置为例，其他平台大同小异。_ 另外如果不一定使用LyX的话，可以参考这里：[http://www.cnblogs.com/rockcode/archive/2011/08/06/2129561.html](http://www.cnblogs.com/rockcode/archive/2011/08/06/2129561.html "真正的Linux下的中文LATEX解决方案: CTeX + xeCJK + XeTEX") 《Linux下的中文LATEX解决方案》，是使用CTeX+XeTeX宏包实现的。

下文写于2011年，很多内容可能已经过时（我自2014年以后就再没用过LaTeX了）。如果是写论文需要，可以参考下列文章or项目：

1. 朋友reverland发布的[北邮毕业论文LaTeX模板以及使用教程](http://reverland.org/linux/2015/11/29/using-buptgraduatethesis-on-ubuntu/)
2. [清华大学学位论文LyX模板](https://code.google.com/p/thuthesislyx/)
3. [云南大学学位论文LyX模板](https://github.com/cherrot/ynuthesislyx)
4. [华南理工大学LyX与LaTeX学位论文模板](http://scutthesis.googlecode.com/)


## 1\. 安装必要软件包

**Ubuntu用户可以直接在源里安装，当然你也可以自己编译LyX并且手动安装最新版的TexLive，从源里安装LyX的话会自动安装源里的TexLive2009。**

1.  在软件中心搜索LyX，点击安装～ 就这么简单！当然了，下面的附加组件也许有些对你有用，参照软件说明选择安装即可。
2.  在软件中心搜索texlive-xetex，点击安装。

## 2\. 配置LyX

第一步：新建一篇文档，然后在菜单栏依次点击 文档——首选项，切换到“语言”，设置语言为英语，编码为其他 Unicode (XeTex) (utf8)

第二步：切换到“字体”选项，勾选 Use non-TeX fonts (via XeTeX/LuaTeX)。如果第一步没有做的话，你会发现，Use non-TeX fonts (via XeTeX/LuaTeX)是灰色不可选的。 另外，如果之前使用过CJK宏包，Use non-TeX fonts (via XeTeX/LuaTeX)可能无法勾选，此时，在LyX菜单上点击工具==>重配置就好了。 

第三步：切换到PDF Properties选项，勾选`使用hyperref`，下方的附加选项记得写上`unicode=false`，这样生成的书签就不会出现乱码了。在书签选项卡上全部打钩，级别设为3

第四步：选择Output选项，设置输出格式。将默认输出格式设置为PDF(XeTeX)

第五步：设置LaTeX序，添加如下代码：

```
\usepackage[BoldFont,SlantFont,CJKnumber,fallback]{xeCJK}%使用TexLive自带的xeCJK宏包，并启用加粗、斜体、CJK数字和备用字体选项
\setmainfont{DejaVu Serif}%设置西文衬线字体,{}中是字体名,可更换,下同
\setsansfont{DejaVu Sans}%设置西文无衬线字体
\setmonofont{DejaVu Sans Mono}%设置西文等宽字体
\setCJKmainfont{Adobe Song Std}%设置中文衬线字体,若没有该字体,请替换该字符串为系统已有的中文字体,下同
\setCJKsansfont{Adobe Heiti Std}%中文无衬线字体
\setCJKmonofont{WenQuanYi Micro Hei Mono}%中文等宽字体
%中文断行和弹性间距在XeCJK中自动处理了
%\XeTeXlinebreaklocale “zh”%中文断行
%\XeTeXlinebreakskip = 0pt plus 1pt minus 0.1pt%左右弹性间距
\usepackage{indentfirst}%段落首行缩进
\setlength{\parindent}{2em}%缩进两个字符
```

关于什么是衬线字体，什么是无衬线字体，可以参考《XeTeX / LaTeX 中文排版之胡言乱语》。我使用了Adobe的免费中文字体（Windows下安装Adobe Reader时会自动安装），可以[在这里](http://ishare.iask.sina.com.cn/f/15105086.html)下载到。如果需要使用Windows下的常用字体，可以在[http://scutthesis.googlecode.com/files/winfonts.zip](http://scutthesis.googlecode.com/files/winfonts.zip) 下载。解压到~/.fonts文件夹后， 再在终端运行fc-cache命令，刷新字体库。 若用fc-list :lang=zh命令，可以到所安装的中文字体（这些字体应该都可以在LyX中使用，直接将名字替换掉上述代码中的字体字符串即可）。 另外说明一下，这个链接其实就是一个好人贡献的华南理工大学LyX与LaTeX的论文模板的项目，项目主页：[http://scutthesis.googlecode.com/](http://scutthesis.googlecode.com/)。

## 3\. 更多设置

在进行了如上设置后，你可能会发现，编辑的文档中如果含有自动编号的图表时，输出的图表编号是英文的，这是因为在我们之前的设置中把语言设置成英文的缘故（而设置成中文是通不过编译的），在《LyTeX中文帮助文档》中找到了办法，使用\renewcommand命令即可办到——将下面的内容添加到LaTeX序中即可将摘要、目录、图表等标记全部转换为中文：

```
\renewcommand\arraystretch{1.2}%1.2表示表格中行间距的缩放比例因子(缺省的标准值为1),中文需要更多的间距
\renewcommand{\contentsname}{目录}
\renewcommand{\listfigurename}{插图目录}
\renewcommand{\listtablename}{表格目录}
\renewcommand{\refname}{参考文献}
\renewcommand{\abstractname}{摘要}
\renewcommand{\indexname}{索引}
\renewcommand{\tablename}{表}
\renewcommand{\figurename}{图}
\renewcommand\appendixname{附录}
\renewcommand\partname{部分}
\renewcommand\today{\number\year年\number\month月\number\day日}
```

**注意，如果放在LaTex序中不起作用的话，那么需要把这一部分放到 \begin{document}后面，即在正文开头插入Tex代码（Ctrl+L），然后将上面内容粘贴进去**。 如果想要修改公式、图表等项目的编号样式，那么需要用到`\numberwithin`命令，比如我要让公式按章节编号，图片按子章节编号的话，把下面的内容添加到LaTeX序中：

```
\numberwithin{equation}{section}%设置公式按章节进行编号
\numberwithin{figure}{subsection}% 图片按子章节编号
```

另外，使用此命令需要用到AMS宏包，确保在文档首选项中的 Math Option中勾选`使用AMS数学包`（而不是`自动使用AMS数学包`）。

## 4\. 强烈推荐三篇教程

《LyTeX中文帮助文档》、xeCJK官方文档和《XETEX / L TEX 中文排版之胡言乱语》 其实本文中的很多内容也是从这些文档中找到答案的，尤其是《LyTeX中文帮助文档》，新手一定要读一读。 如果这两篇教程和本文所涉及到的字体资源用户可以自行替换，如果本文涉及的字体或其他资源您找不到的话，欢迎留言告诉我。

## 6.ChangeLog

*   2012-01-13 添加了第三部份：更多设置，对修改公式、图表的编号语言和样式等进行了介绍。
*   2012-03-02 如果之前使用CJK宏包，使用系统字体的设置可能会出问题，补充了对该情况的说明。
*   2012-03-02 更新了对字体设置的说明。
*   2012-04-26 经Adam8157提醒，如果直接使用XeTeX的话，unicode=false需要去掉，但LyX（起码是我的LyX）必须使用此选项才能避免目录乱码。另外，中文断行和弹性间距在XeCJK中已自动处理，无需再设置

P.S. 如果觉得XeCJK用起来还是很麻烦，你也许应该考虑一下CTeX宏包，它是对XeCJK的进一步封装；或者使用CTexLive套装，可以下ISO镜像文件安装。 
PP.S. 最后分享一下我的LaTeX序言吧： 

```
%中英文混排设置%
\usepackage[BoldFont,SlantFont,fallback,CJKchecksingle]{xeCJK}
\setmainfont{DejaVu Serif}%西文衬线字体 DejaVu
\setsansfont{DejaVu Sans}%西文无衬线字体
\setmonofont{DejaVu Sans Mono}%西文等宽字体
\setCJKmainfont{Adobe Song Std}%中文衬线字体 Adobe宋体
\setCJKsansfont{Adobe Heiti Std}%中文无衬线字体 Adobe黑体
\setCJKmonofont{WenQuanYi Micro Hei Mono}%中文等宽字体 文泉驿等宽微米黑
\punctstyle{banjiao}%半角字符</code>

%其他中文设置 使用XeCJK无需再设置中文断行和弹性间距%
%\XeTeXlinebreaklocale “zh”%中文断行
%\XeTeXlinebreakskip = 0pt plus 1pt minus 0.1pt%左右弹性间距
\usepackage{indentfirst}%段落首行缩进
\setlength{\parindent}{2em}%缩进两个字符

%编号语言、样式设置%
\numberwithin{equation}{section}%设置公式按章节进行编号
\numberwithin{figure}{section}% 按章节编号
%\numberwithin{figure}{subsection}% 按子章节编号

%以下内容（\renewcommand）可能需要放置在\begin{document}之后才能起作用
\renewcommand\arraystretch{1.2}%1.2表示表格中行间距的缩放比例因子(缺省的标准值为1),中文需要更多的间距
\renewcommand{\contentsname}{目录}
\renewcommand{\listfigurename}{插图目录}
\renewcommand{\listtablename}{表格目录}
\renewcommand{\refname}{参考文献}
\renewcommand{\abstractname}{摘要}
\renewcommand{\indexname}{索引}
\renewcommand{\tablename}{表}
\renewcommand{\figurename}{图}
\renewcommand\appendixname{附录}
\renewcommand\partname{部分}
\renewcommand\today{\number\year年\number\month月\number\day日}
```
