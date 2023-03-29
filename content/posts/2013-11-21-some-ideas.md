---
categories: Thinking
comments: true
date: "2013-11-21T00:00:00Z"
title: 放慢脚步回首过去一个月
---
自上篇文章到现在差不多快一个月了，也因各种琐碎的事情没有闲下来构思一篇博文，顿时有一丝的罪恶感。吃完晚饭洗完澡坐在电脑前突然有种茫然的感觉，上了会儿高端大气的知乎浏览了几个帖子，本想着改下代码，结果发现也打不起精神，便趁此机会来码字来寻求心灵片刻的安宁吧。

首先，说说最近的劳动吧，虽然没啥成果。最近教研室事不多，刚好师姐有个同学需要找人帮忙改个项目，便答应帮忙(当然是有酬劳的，不然也不会闲到改.net的代码，虽然目前还没有谈具体的数字)。其实也不在乎都多少，答应干此活也只是为了积累经验而已，方便日后找工作，同时也算是练手。拿到代码后，我顿时有种欲哭无泪的感觉，代码逻辑及结构之混乱超乎我想象，可以总结为以下几点：

<!--more-->

- 结构混乱，虽然有分层，从下到上大致有DAL、BLL、WebService和Web这几层，然后还有一个Common，里面大致是定义一些通用的操作供DAL调用。但是后来发现代码的逻辑几乎都是在Web层直接进行的，询问得知此系统是在它们以前的OA系统基础上改的，所以保留着以前OA的那种分层结构，只是新开发时没有按照以前的那种分层走。

- 命名**极其**糟糕。之所以突出*极其*二字，是因为它的命名真的已经到了惨不忍睹的地步，有种想撞墙的感觉。我真的怀疑写代码的人知道常见的几种命名法则不，如驼峰命名法、下划线命名法、Pascal命名法和匈牙利命名法(在此提匈牙利命名法，不知道会不会遭人鄙视，之前看到网上已经有太多的人吐槽，连Linus大神也吐槽过)。我个人还是比较青睐于驼峰命名法和下划线命名法，看着很优雅。接着不得不吐槽变量的命名了，一会儿用英文命名，一会儿用对应的拼音，混在一起感觉有点奇葩感。我个人是不太喜欢用拼音，即使不了解，也会用工具将中文翻译成对应的英文再做命名，不知道这算不算强迫症。

- 代码逻辑凌乱。逻辑与逻辑间缺少空行分隔，所以有种目不忍视的感觉。还有比较重要的一点是代码重复性太高。比如说省市县三级下拉菜单，很多页面都会用到，但是每个页面都会重复性地写这些代码。如果有机会，我真的会问写此代码的人懂不懂什么叫DRY(Don't Repeat Yourself)，虽然此思想是学习Ruby过程种了解到的，但我觉得可以用于任何语言，放置四海而皆准。毕竟重复性的工作会导致代码的耦合性高，如果今后需要变动，那么每个引用此代码的地方都得做相应修改。

- 注释过少。尤其是功能性的注释太少，虽然通过阅读代码最终也能了解，但能不看内部实现便不看，毕竟看代码还是需要花费一定时间的，而且理解可能还会有偏差，所以还是很有必要对代码做相应的功能性注释，不仅方便他人，也方便自己。所以要了解一个人的编码水平，通过观其代码便能略之一二。

- 没有使用ORM。我个人也感觉这算是一个弊端，毕竟对于应用开发者来说，效率至上(指开发效率)。虽然直接写Sql语句可能效率更高，但是为此付出的开发时间成本则是不扉，在对性能要求不是很极端的情况下也就显得有些得不偿失。ORM(Object Relationship Mapping)不仅使开发更高效，而且代码也更易阅读，更易维护。反正我了解的应用开发一般都会用到ORM,如.net的EF(Entity Framework),Java的Hibernate、Mybatis等，Ruby的ActiveRecord等。其他语言也应该有相应的ORM,不过不了解，我了解的就是这三门语言，所以也便能说出一二。

说了这么多弊端，不知道是不是由于我平时的开发一般都是MVC模式，所以对其他的开发模式看不习惯所致。不管怎样，还是得硬着头皮改完吧，既然承诺于人，便得有始有终，这也算是我一贯的办事风格吧。

其次，由于老师让看Hadoop,所以便时不时的看看云计算和Hadoop这块，算法还真不是我的强项，看到朴素贝叶斯和KNN算法，真的是看不下去，尤其用MapReduce实现。MapReduce程序真的是和一般的代码差别有点大，把握不住运行的脉络，一旦看不清脉络，便有种管中窥豹的感觉。身边也没有搞Hadoop的人，想找个人聊聊都困难。网上吧，提的问题太过白痴，也往往遭人鄙夷，从此便被无视。这算是如今最头疼的事了。

最后，说说加入Rubyists的情况吧。2012年，Oschina在西安举办OSC，作为开源的爱好者与拥护者便跑去参加了。正是这次机缘巧合接触了Ruby，接着便花了一些时间去了解Ruby和Rails。在RubyChina社区上又无意看到了一个关于西安Rubyists线下活动的召集贴，便报名参加了，从2013.7第一次参加到现在也差不多5个月了，也就是5次活动(每月一次),要说最大的收获应该就是眼界的拓宽，对IT行业有了一个更深的认识，对新技术也有了一点了解，如angularjs,远程工作的一些工具，同时也更加坚定了自己对Git/GitHub(以前一直都是一个人在使用，身边的同学还没发现使用Git，用了svn都算不错了)及Vim的使用。Vim以前也只是会一些基本操作，配置都是拷贝网上现成的，到现在的熟练运用，可以说也算是有质的飞跃了。虽然目前一直在Rubyists中一直处于索取的状态，不过相信迟早也会奉献的，可以讲讲Vim的操作、配置及常用插件等。

PS:刚在知乎上看到某人专栏上的一篇文章，觉得有部分写的不错，便搬过来吧，当然不是完全照搬，我会结合我自己切身的经历用自己的语言描述出来。大致如下：

由于我所学的专业为计算机，所以我对这一领域有找寻答案的能力，因此在碰到问题时能很快的定位答案，予以解决，真的可谓时见招拆招。如果换作非计算机专业的人来说，可能显得稍微吃力。当一个人想知道[门]背后有什么的时候，他需要的只是开启门的钥匙，而这个钥匙刚好在我手里，因为即使我也不知道门背后有什么，但我却能够将门打开，让大家看到这门背后到底是什么。而帮人解决问题的过程则让我了解门背后的东西，还收获了开启各种门的方法。

我以前老喜欢上网浏览信息，比如说CSDN、新浪微博、RubyChina等，但慢慢这种趋势有所下降。因为我感觉看的东西再多，慢慢也便淡忘，从上面所了解的知识再丰富、再专业，也不过是碎片化的，不会提供一个系统的知识，所以也便不可能形成完整的知识结构。没有一个完整的知识结构，知识储备便很难有质的突破。所以我也慢慢更多地转向了书本的学习，毕竟从书本上看的东西还是比较系统化。

最近在阅读的一本书是《高效程序员的45个习惯》，真心值得一读。身处碎片化的时代，能够静下心来读一本书真的很不容易。希望接下来的日子里能将看微博、豆瓣等时间更多的分给读书。不知不觉已经10点多了，就这样断断续续的竟写了三个小时，到此结束吧，也算是最自己近一个月来的总结。不认识过去，便看不清未来。