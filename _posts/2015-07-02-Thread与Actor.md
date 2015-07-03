---
layout: post
title: Thread and Actor
date: 2015-07-02
categories: scala
---

Thread与Actor是两种完全不同的多线程设计思路，其中的核心关键在于，是否使用共享内存（Thread角度），或者说，是否使用消息传递数据（Actor角度），实际上前面两句话的意思在这里是一致的，改变线程之间的交互方式，从而引出Thread与Actor两种不同的多线程设计模式。

在平时初学Java多线程或者Windows多线程时，甚至于P-Thread等多线程模型下，思考的方式都是通过共享内存，来打到多线程之间的交互，共享内存包括锁，互斥量，信号量等形式，也有volatile等标识的可见性内存变量，最后还有Future这一较少涉及到的