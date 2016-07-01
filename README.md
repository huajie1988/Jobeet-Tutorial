# Jobeet-Tutorial

基于Symfony2的Jobeet官方实战课程个人汉化

目前状态：WIP

进度：汉化 100% 第一次校对 20% 润色 0% 备注补充 20%

源文本取自：

前14篇来自于[JOBEET TUTORIAL WITH SYMFONY2](http://www.ens.ro/2012/03/21/jobeet-tutorial-with-symfony2)

后5篇来自于[这里](http://intelligentbee.com/blog/category/php-and-symfony)

其中有部分在intelligentbee网站中补充的新内容(比如`Composer`)会适当补充到前14篇中


# 目录
1.[总览](01_General.md)  
2.[Symfony2 Jobeet Day 1: 开始项目](02_Starting_up_the_project.md)  
3.[Symfony2 Jobeet Day 2: 关于项目](03_The_Project.md)  
4.[Symfony2 Jobeet Day 3: 数据模型](04_The_Data_Model.md)  
5.[Symfony2 Jobeet Day 4: 控制器与视图](05_Controller_and_the_View.md)  
6.[Symfony2 Jobeet Day 5: 路由控制](06_The_Routing.md)  
7.[Symfony2 Jobeet Day 6: 深入Model](07_More_with_the_Model.md)  
8.[Symfony2 Jobeet Day 7: 玩转分类页](08_Playing_with_the_Category_Page.md)  
9.[Symfony2 Jobeet Day 8: 单元测试](09_The_Unit_Tests.md)  
10.[Symfony2 Jobeet Day 9: 功能测试](10_The_Functional_Tests.md)  
11.[Symfony2 Jobeet Day 10: 表单](11_The_Forms.md)  
12.[Symfony2 Jobeet Day 11: 测试你的表单](12_Testing_your_Forms.md)  
13.[Symfony2 Jobeet Day 12: Admin Bundle](13_The_Admin_Bundle.md)  
14.[Symfony2 Jobeet Day 13: 安全](14_Security.md)  
15.[Symfony2 Jobeet Day 14: 订阅](15_Feeds.md)  
16.[Symfony2 Jobeet Day 15: Web Services](16_Web_Services.md)  
17.[Symfony2 Jobeet Day 16: 邮件](17_The_Mailer.md)  
18.[Symfony2 Jobeet Day 17: 搜索](18_Search.md)  
19.[Symfony2 Jobeet Day 18: AJAX](19_AJAX.md)  
20.[Symfony2 Jobeet Day 19: 国际化和本地化](20_Internationalization_and_Localization.md)  


这里随便写一些东西吧，我当时入门Symfony的时候其实也算挺晚的了，当时最新版是2.7。。。当初原本是计划学Laravel 5然后以此搭一个自己的博客，然而那时候阿里的虚拟主机只支持PHP 5.3。。。迫不得已就上了这个贼船╮(╯-╰)╭ 

不过也庆幸之前Laravel的一些基础对我帮助挺大，尤其是知道了Composer，当时第一次发现原来PHP可以这么玩，那震撼就跟在刀耕火种时代突然看到蒸汽机一样。

后来就开始一步步的学，一步步的摸索，各种问题还是很多的，尤其是当时国内那少的可怜的教程。。。话虽如此不过好的教程还是有的，这里主要推荐两个：

一个是博客园网友Seekr的几篇文章，地址是[这里](http://www.cnblogs.com/Seekr/tag/Symfony2/)
另一个是慕课网的[《洪大师带你解读Symfony2框架》](http://www.imooc.com/learn/244)。这是我比较推荐的，因为视频学习起来会轻松不少，主要是不会被一大团文字搞得没心情>_<。

基本上你看完视频之后就对Symfony有了一个比较大体的了解了，那么再结合文章和这个教程搭建一个网站的话就会事半功倍了。

话说一般如果你已经看了这个教程，那么理论上应该已经选定了某个项目或出于某种原因来学习Symfony了，不过我在这里还是简单说说之所以要选Symfony的原因吧。

首先第一点就是学习成本，有人肯定会说这学习成本明显更高啊，学起来也累，但如果你真的学习了之后会发现其实这个学习成本反而降低了。就像你先学会了C，再看其他语言是不是就简单多了？举个实际点的例子好了，当初我把Symfony视频教程看完之后，虽然只是一知半解，但当我实际项目要用到Yii2的时候发现完全是手到擒来，哪怕只是我第一次用。你会发现现代的优秀框架思路可能都是大同小异的，区别可能只有少许的语法和模板上不同，这显然大大降低了学习成本。

第二点就是便于后期维护和扩展，当然这并非Symfony独有的，只要这个框架支持composer一般都能支持扩展，这也意味着你可以不用反复造轮子，只要有能用的引用安装就行了，完全告别原来那种到处找类库的烦恼了。

第三就是足够强大，几乎包含了你开发生命周期所需要的一切，web开发工具栏，高度的封装，完善的测试，还有强大的命令行等等，就和Emacs一样，你以为它只是个框架，其实它是伪装成框架的IDE o(*≧▽≦)ツ┏━┓

当然或许有人可能会有不同看法，这很正常，毕竟不能强求，但如果你用了，我想至少不会太失望。

说完了优点，自然也要说说缺点，基本上最大的问题就是学习曲线太陡，到现在我也只能说我是会用，不敢说掌握，精通那就更别提了。很简单，首先本身的语法你需要学习，思路需要理解，接着可能还有service、event这类你要懂吧，这还没完，别忘了最重要的Doctrine和Twig也要学，甚至到后来你可能要自定义一个表单的field，那就很痛苦了(然而这种时候我会改成ajax～(▔▽▔～))。

当然就象我之前说的，虽然你学起来难，但收获是与付出成正比的，既然如此，何不试试？

最后也稍微聊聊我为什么汉化吧，其实我原本是没打算汉化这个的，看完就好了(其实主要原因还是英语太渣。。。不过感谢这个世界上有个东西叫必应翻译XD)。然而有次基友说他准备汉化TypeScript的Handbook，于是我就说为什么不直接翻一个国外的实战项目比较好呢？于是他说那你为什么不试试翻一个呢？就这样我开了这个坑。。。然后看着版本从2.7一直升到了3.1 ╮(╯▽╰)╭

另外之所以前14章和后5章不是一起的是因为我当时以为作者只有14章。。。毕竟谷歌出来Jobeet之后推荐的就是这前14章的地址，直到有一天我突然想看看作者其他的文章时才发现后来换了地方Σ(っ °Д °;)っ而当时我已经完成了前8章的内容了，好在后来看看差别不是特别大，因此就决定以前14章为蓝本，加入新的内容为辅的形式。当然也是有一些修改的，不过基本原则是从新不从旧，从简不从繁。当然也适当的加入和删除了一点，比如第一章我将新版的服务器配置的内容都删除了，主要是考虑到第一原主版本没有，第二是作为一名阅读本教程的WEB开发人员，最基本的服务器安装默认应该会了，那就没必要把这些也翻译出来。

好了说了这么多，我估计都没人看到这儿(T-T)。

好了废话不多说，[开始我们的学习之旅吧](01_General.md)