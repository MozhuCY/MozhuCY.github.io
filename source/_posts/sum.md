---
title: 对于CTF和漏洞挖掘的一些浅显看法
date: 2020-03-17 10:27:50
tags: 杂七杂八
---

最近在转型实战,在这段期间也体会到了实战与CTF的不同.就像之前学长说过的一样,CTF并不是全部,对于二进制方向来说,CTF中所用到,所学到的东西,更多的是一种工具.

逆向方向在实际的漏洞挖掘中,仅仅是一种辅助手段. 在实战中,逆向是对于一些闭源软件的漏洞挖掘的入口点.只有通过逆向,才能整理出代码的逻辑,结合逆向的知识,一些代码阅读量对于某些闭源软件的实现进行理解. 包括调试,插桩,模拟执行,行为检测,网络监控,其实都是对于不同目标的不同分析方法.CTF中我们了解了这些,但是并不意味着在实战中,我们可以灵活的应用这些方法.所以在转型的期间,就要训练这种能力,改掉在CTF比赛中的一些习惯,区分开"虚拟与现实".

而PWN方向,在CTF中更多的考察应该在于如何利用.据我观察,在转型的过程中,基本上所有的PWN选手对于一个已知漏洞的利用,重写POC,都是不成问题的.在CTF中,我们学到了一些内存管理,对于底层有了了解,从kernel到VM,其实都是对于一些机制,一些很简单的原理的一些理解,在实战中,这些浅显的道理是不够组建起一个完整的知识体系的.所以这也是为什么CTF一场比赛只是几天的时间,而现实的一个漏洞往往需要几个月甚至几年.

可以看到一些高质量的CTF比赛排名靠前的战队,其战队成员也一定是在其领域有很深的造诣的.因为在高质量的CTF比赛中,考察的更多的是更为现实的环境,以及对于综合能力的运用.

所以漏洞挖掘其实是一个全新的领域,CTF中学到的东西只是让我们有了开始漏洞挖掘的基础,就相当于读汇编相对于逆向一样,后面的东西还是需要慢慢的学习的.
