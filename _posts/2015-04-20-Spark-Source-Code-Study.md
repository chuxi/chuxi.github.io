---
layout: post
title: Spark Source Code Study
date: 2015-04-20
categories: spark
---

之前自己也修改过Spark的源代码，但实际上只是外层的一些根据接口做的小调整而已，所以对Spark的运行机制没有什么深入的理解。而自己对于Java和Scala也是极其粗浅的理解，所以看着Spark的源代码很难理解说，作者为什么要这样写这一段代码。所以在正式开始调试跟踪Spark代码之前，自己拿着《Thinking in Java》和《深入理解Java虚拟机》两本书狠狠地啃了一番，对于对象信息，IO接口，序列化等概念也进一步学习掌握，此外还有concurrent和Actor多线程机制进行了深入的学习，至此，自己一共花费两周左右时间，可以开始看懂Spark源代码了。但是的确磨刀不误砍材工，自己在阅读Spark源代码时也就能够顺利看懂其中很多细节上的设计，也是一边看一边对Spark的那些作者们大吐敬仰之情。

### Spark Job Process
