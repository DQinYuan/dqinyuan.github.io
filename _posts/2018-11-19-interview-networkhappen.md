---
layout: post
title: 大厂常见变态面试题解析（Java方向）（一）从网址输入浏览器到网页呈现在屏幕上的过程
date: 2018-11-19
categories: java
tags: java 并发
cover: 
---



# 引言

在大厂面试过程中经常会遇见一些“变态”的题目，这些题目如果没有提前准备，一时间还真无法下手，这个系列文章我就想总结总结这些经常会被问到的“变态”题目的应对策略。



# 总

“说说从网址输入浏览器到网页呈现到屏幕上，这个过程中发生了什么？”



在大厂面试中经常会遇到这个问题，考得就是计算机网络的基础是否扎实，思维是否开放，然而面试过程中尝尝一紧张，就只记得一个HTTP请求了，然后面试官就会很不满意，觉得面试者网络的基础很差，特地写这篇文章总结了一下。



这个过程说起来复杂，但是概括起来说就是以下四个步骤：

- DNS解析
- HTTP请求
- 服务端处理
- 浏览器渲染



# DNS解析



首先就是要将域名映射成其对应的IP地址，细节步骤如下：



### 1、去浏览器缓存中寻找映射

浏览器会缓存这种映射关系，如果在浏览器缓存中存在的话就优先在浏览器缓存中查找



### 2、去操作系统缓存中寻找映射

操作系统也会缓存DNS映射，这一步需要设计到系统调用



### 3、去路由器缓存中寻找映射

路由器中同样有缓存



### 4、去本地域名服务器（local name server）中寻找

本地域名服务器往往就是你在操作系统中配置DNS Server:

​    





# 参考文献

1. 国外大佬的博客： http://igoro.com/archive/what-really-happens-when-you-navigate-to-a-url/
2. 《计算机网络》（第五版，谢希仁著）
3. ETAG：https://www.cnblogs.com/softidea/p/5986339.html
4. Long Polling：https://www.jianshu.com/p/d3f66b1eb748?from=timeline&isappinstalled=0
5. BigPipe：http://taobaofed.org/blog/2015/12/17/seller-bigpipe/