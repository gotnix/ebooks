## SICP 网上公开的课程

[TOC]

### 零 TLNR.

如果你打算完整自学，最高效的组合是：

1. 看 John DeNero 的视频快速过一遍概念。
2. 去做 CS61A 官网的作业（Lab/Homework）。
3. 如果遇到某个概念（比如闭包或宏）死活不理解，切到 Paul Hilfinger 或 2019 Summer 的对应章节听不同人的解释。

这几位讲师虽然风格不同，但他们教的都是同一个“灵魂”：SICP 的抽象哲学。 无论看哪一个，你最后都要面对那个用 Python 写 Scheme 解释器的魔鬼级 Project，那是衡量你是否真正通过 61A 的唯一标准。

由于 SICP 的精髓在于亲手写代码，国内最好的学习路径其实是：
看 B 站精翻的 John DeNero 2024 版（也就是你之前给出的链接），同时参考 裘宗燕老师的中文译本 查缺补漏。如果你在学习过程中对“环境模型”感到头大，去 Source Academy 玩一玩那个可视化的 JavaScript 版。

如果学习过程中发现基础知识不足，先学习哈弗大学的CS50 《计算机科学导论》，更多关于学习路径的内容请看 **CS自学社区** 提供的 [路线图](https://www.learncs.site/docs/roadmap) 。
- https://www.bilibili.com/video/BV1Rm421V7zw
- https://cs50.harvard.edu/summer/2023


### 一 UCB 的 CS61A
#### 1.1 LISP 版本
1. [目前互联网上最早的CS61A课程，2008 FA，CS61A，LISP](https://www.bilibili.com/video/BV1cM4y1R7iE/)
    1. 视频内部的演示文稿信息：
        > The Structure and Interpretation of Computer Programs  
        > Computer Science 61A  
        > Brian HARVEY  
        > functional programming Ⅰ  
        > Auguest 27, 2008
    2. 视频页面的描述信息：
        > https://archive.org/details/ucberkeley_webcast_itunesu_354818329  
        > 01\. 2008-08-27 - functional programming I
        >
        > Addeddate 2018-02-07 07:05:39  
        > Identifier ucberkeley_webcast_itunesu_354818329  
        > Itunes_copyright Copyright UC Regents 2008  
        > Itunes_id 354818329  
        > Itunes_name Computer Science 61A|Fall 2008|UC Berkeley  
        > Itunes_url https://itunes.apple.com/us/itunes-u/computer-science-61a-fall-2008-uc-berkeley/id354818329?mt=10
        >
        > Rss_title Computer Science 61A|Fall 2008|UC Berkeley  
        > Rss_url https://deimos.apple.com/WebObjects/Core.woa/GetRSS/berkeley.edu-dz.3298700603.03298700605?U=http%3A%2F%2Fwbe-itunes.berkeley.edu%2Fmedia%2Fcommon%2Frss%2Fcomputer_science_61a_Fall_2008_Video__itunes.rss 
        > 
        > Scanner Internet Archive Python library 1.7.6

#### 1.2 Python 版本

##### 1.2.1 教材与代码实践

1. 现代适配教材 (Python 版)
    - [Composing Programs](https://www.composingprograms.com/)：John DeNero 教授为 CS61A 量身定制的在线教材。它以 Python 为媒介，完美重构了 SICP 的核心思想，是目前该课程的首选主教材。
    - [SICP Python 描述（中文版）](https://wizardforcel.gitbooks.io/sicp-py/content/)：此版本即为《Composing Programs》的中文翻译版，极大降低了中文读者的入门门槛。
    - [CS61A Online Textbook](http://www-inst.eecs.berkeley.edu/~cs61a/sp12/book/)：伯克利官方早期的在线版本（注：目前(2026/03)该站点需要校方账号登录方可访问）。
    - 离线方案：如需在 Kindle 或移动端阅读，建议从 [安娜的档案](https://zh.annas-archive.gd/md5/d9634d95514062df53cc018bb7f732ff) 获取该书的 Epub 格式进行离线自学。《SICP Python 描述（中文版）》是通过 GitBook 发布的，可以把 [Git 仓库](https://github.com/wizardforcel/sicp-py-zh/blob/master/1.1.md) 克隆到本地阅读 Markdown 文件。

2. 经典原著参考 (Lisp/Scheme 版)
    - [SICP, 2nd Edition (Official Web)](https://sarabander.github.io/sicp/html/index.xhtml#SEC_Contents)：由 Harold Abelson 等人编写的原著第二版在线 HTML。虽然现代 CS61A 切换到了 Python，但原著关于“抽象”的哲学讨论依然是不可逾越的经典。

3. 辅助工具与工程实践
    - [项目代码库 (Projects)](https://www.composingprograms.com/projects.html)：包含课程中著名的 Hog、Maps、Ants 以及 Scheme 解释器等 Project 的源码框架，是衡量学习效果的硬指标。
    - [Online Python Tutor (可视化)](https://www.composingprograms.com/tutor.html)：极力推荐。该工具能动态展示代码执行过程，特别是在理解递归、闭包（Closures）和环境框架（Environment Frames）时具有不可替代的直观效果。

##### 1.2.2 有中文字幕的视频 Bilibili 视频

1. [【完结】【CS61A精翻双语·英文原声】伯克利大学《计算机程序的结构与解释》(2024)](https://www.bilibili.com/video/BV1sy411z7nA/)
    1. 主讲人：John DeNero & Anant Sahai / Rao
    2. 核心特点：工业感与快节奏：DeNero 是这门课 Python 化的奠基人，逻辑如手术刀般精准。没有任何废话，逻辑极其严密，动画演示（Python Tutor）非常多。它更像是一套视频版的“交互式教材”。
    3. DeNero 的魅力：他是谷歌翻译团队的初创成员，他的讲课风格极其现代，重点在于如何用代码构建复杂的系统。
    4. 最新的 Python 实践：2024 版会包含 Python 最新的语法特性，以及对 AI 时代编程思维的微调。
    5. Rao 的加入：Rao（通常指 Anant Sahai 等教授协同）通常会带来更多关于算法效率和严谨数学逻辑的视角。
    6. 推荐人群：追求极致的课程设计、想要最丝滑的 Python 学习曲线、并希望跟上伯克利最新教学大纲的同学。
    7. 完整度： 它是高度精炼的。这些视频是 DeNero 专门为“翻转课堂”录制的。每个视频通常只有 5-15 分钟，只讲一个核心概念（如“Environment Diagrams”或“Recursion”）。 
    8. 缺失部分： 它缺少线下课（Live Lecture）中的学生提问、即兴的代码 Debug 演示、考试技巧指导以及针对当季作业的特别提示。

2. [【计算机程序的构造和解释】精译【UC Berkeley 公开课-CS61A (Spring 2021)】-中英双语字幕](https://www.bilibili.com/video/BV1v64y1Q78o/)
    1. 主讲人： Paul Hilfinger & Pamela Fox
    2. 核心特点：硬核与底层，Hilfinger 是伯克利的传奇老将，风格非常“硬核”，更偏向编译器和底层细节。更有“临场感”。你会看到教授如何当场写代码，如何解释学生在 Zoom 聊天框里提出的各种刁钻问题。
    3. Hilfinger 的“杀伤力”：在伯克利，Hilfinger 以“虐生”著称。他的课程虽然也用 Python，但他会比 DeNero 更多地提及内存模型、指针概念（即使是 Python 里的引用）以及代码的机器级表现。
    4. Pamela Fox 的互补：Fox 教授的风格非常亲民、细致，擅长通过可视化工具和生动的例子把 Hilfinger 那些硬核的概念讲通。
    5. 怀旧与厚度：虽然是 2021 年，但 Hilfinger 的讲课风格更接近 90 年代那种“老派程序员”的严谨感。
    6. 推荐人群：如果你觉得 DeNero 讲得太快、太“精英化”，或者你想听听老一辈教授对编程本质的独特理解，选这个版本。
    7. 完整度： 这是全过程实录。每节课 50 分钟到 1 小时。
    8. 缺失部分： 节奏可能比 DeNero 的微课慢，且由于是直播录屏，视频质量和画面切换可能不如专为录播设计的微课版本。

3. [CS61A SICP，2019 SU，双语字幕](https://www.bilibili.com/video/BV1nJ41157p6/)
    >
    >2019 SU 学期的课程，讲师/教授有 Chris Allsman、Tiffany Perumpail、Alex Stennet
    
    暑期课程（Summer Session）在伯克利是一个非常特殊的时期，Chris Allsman、Tiffany Perumpail 和 Alex Stennet 当时都是优秀的研究生讲师（GSI）。课程特点如下：

    1. 高强度、短周期  
    暑假班是将常规 15 周的课程浓缩到 8 周内完成。这几位讲师的视频通常节奏极快。
    2. 更贴近学生视角  
    由于讲师本身就是刚带完几届助教的研究生或高年级学生，他们非常清楚学生最容易在哪个坑里掉下去。他们的讲解往往更具实操性，会分享很多他们自己当年学这门课时的“   避坑指南”。
    3. 重点更突出  
    因为时间紧迫，暑期版会去掉一些非核心的延展内容，更加聚焦于 **大作业（Projects）和考试（Exams）** 本身。如果你发现 DeNero 或 Hilfinger 讲得太深奥，    看这些学长讲师的版本往往能让你瞬间“接地气”。

### 二 NUS CS1101S

#### 2.1  与 UCB CS61A 相比的核心差异点

1. 编程语言的哲学
    - CS61A (Python 版)：
        - 工业界结合更紧密：使用 Python 意味着你可以直接接触到现实世界最流行的语法和库。
        - 对象模型：虽然也讲函数式编程，但对 Python 的 **对象系统（Object System）和变动状态（Mutable State）** 讲解非常深入。
        - 项目导向：大作业（如模拟游戏、实现解释器）通常规模较大，具有很强的趣味性和工程感。

    - CS1101S (JS 版)：
        - 学术纯粹性更高：虽然用的是 JS，但他们将其裁剪成了 "Source" 语言。它剔除了一些混乱的 JS 特性，强制你用纯粹的函数式思维思考。
        - 环境模型可视化：NUS 专门开发了 Source Academy 平台，能非常直观地展示函数调用栈、环境框架（Environment Frames），这对于理解闭包和作用域极为有效。

2. 教学重点的偏移
    - CS61A 更像是一门 **“通用计算机科学入门”** ：它涵盖了函数式编程、面向对象编程、声明式编程（SQL）以及简单的分布式系统概念。
    - CS1101S 更像是一门 **“编程范式与构造入门”** ：它更贴近 SICP 的原意，花大量篇幅讨论如何通过简单的原始元素构建复杂的抽象，以及元循环解释器（Metacircular Evaluator）的构建。

3. 学习工具与体验
    - CS61A：主要是基于本地环境配置，使用终端、解释器进行自动测试（OK 套件）。这能锻炼你处理真实开发环境的能力。
    - CS1101S：主要使用其在线平台 Source Academy。这个平台带有“魔法感”，比如可以把代码转化为音乐，或者通过可视化工具观察递归树。

#### 2.2 图书和代码：
- 新加坡国立大学 (NUS) [课程官网](https://www.comp.nus.edu.sg/~cs1101s/) 。
- [Structure and Interpretation of Computer Programs, Comparison Edition](https://sicp.sourceacademy.org/)，新加坡国立大学 SIPC JS 对比版的在线教材。
- 由 Source Academy 提供的 playground 作为在线的 IDE：https://sourceacademy.org/playground
