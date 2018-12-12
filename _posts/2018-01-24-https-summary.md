---
layout: post
title: Https协议原理总结
date: 2018-01-24
categories: java
cover: https://img-blog.csdnimg.cn/20181212234503303.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzMjU2Njg4,size_16,color_FFFFFF,t_70
tags: 网络
---


# 说明
　　文章是根据中科大郭燕老师的信息安全实践课程的Lecture1 Https的内容外加自己的理解改写的。

# HTTPS的产生

众所周知，HTTP协议存在非常大的安全隐患，首先它在网络上传输明文，这样导致其传输的内容非常容易在网络上被人窃听；其次，它无法给服务器或者客户端提供有效的身份认证手段。因此而诞生了HTTPS协议弥补这些不足。

HTTPS在TCP与HTTP协议之间添加了一个SSL层，SSL层允许服务器认证，客户端认证与加密通信。

# SSL层的工作原理：非对称加密（RSA算法）

RSA算法的直观理解：
　　所谓非对称加密就是有一对密钥，我们称之为公钥和私钥，公钥加密可以由私钥解密，私钥加密可以由公钥解密，并且由公钥推知私钥是不可计算的，反之亦然。这个是如何做到的呢？举个直观的例子：
　　假如我有一个三位数"123"需要加密，加密密钥为"11"，之后进行"加密"操作，即将这两个数相乘，得到的加密后的值为"1353"，收到这个加密信息后，接收方要进行解密，解密密钥是"91"，进行"解密"操作，解密操作也是相乘，将"1353"乘"91"后得到"123123"，我们只要取其低三位就可以得到原文了。刚刚的操作是用"11"加密，"91"解密，如果你尝试一下会发现用"91"加密，"11"解密也是可行的，其实这一切都源于1001可以分解11和91这两个因子所导致。

![RSA直观理解](https://img-blog.csdnimg.cn/20181212234351384.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzMjU2Njg4,size_16,color_FFFFFF,t_70)

　　以上只是对RSA算法的直观理解，真正的RSA算法比这个要复杂，如果想要进一步了解这个算法以及它的证明的话，可以参考博客http://www.matrix67.com/blog/archives/5100
　　一般公钥是公开的，私钥是自己要妥善保管的，这种加密算法出了可以用来进行加密通信外，还可以进行数字签名（即用私钥加密）来进行身份验证。

# HTTPS工作原理示意图：

![https工作流程图](https://img-blog.csdnimg.cn/20181212234503303.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzMjU2Njg4,size_16,color_FFFFFF,t_70)

首先阐清图中几个概念：
 - 证书（certificate）：网站的证书包括网站的公钥，域名，过期时间以及其他相关信息；
 - CA：CA是一个可以信赖的第三方，它自己有一对公私钥，网站可以；
 - 根证书（root certificate）：也就是CA机构的证书

工作流程：在流程中有四个角色，分别是CA机构，浏览器厂商，Web站点和用户浏览器。逐一解释图中的1,2,3标记的过程：

  1. 在CA机构建立之初，它便会将它自己的证书交给浏览器厂商让厂商将这个证书打包在浏览器软件中，之后浏览器厂商再将浏览器分发给用户使用；
  2. Web站点会请求CA机构给自己进行认证，CA机构对该站点进行考察之后同意认证，于是就用自己的私钥给该Web站点的证书进行签名（所谓签名就是用私钥加密）；
  3. 当用户请求Web站点时，Web站点会将它的证书发给用户浏览器，用户浏览器用CA的公钥尝试进行解密，如果能够正确解密的的话，则认可这个网站的安全性，否则的话就会展现出我们在Chrom浏览器中经常看见的"您的连接不是私密连接"页面，如下：


 ![https认证失败画面](https://img-blog.csdnimg.cn/2018121223454236.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzMjU2Njg4,size_16,color_FFFFFF,t_70)

# 一些细节补充：
1.在用户浏览器确认了网站的安全性之后，通信过程的加密是如何进行的？在通信过程中使用的还是非对称加密吗？
　　答：用户浏览器与站点之间会先使用非对称加密商量一个session key，之后的通信就全部使用session key进行对称加密通信了。至于为什么不一直采用非对称加密，是因为对称加密的效率要比非对称加密高。

2.浏览器是如何确认证书是否与网站匹配的？
　　答：浏览器在收到站点证书并且使用CA的公钥成功解密之后，会比对当前浏览器域名框中的域名与站点证书中"使用者可选名称"（如下图中的站点证书的可选名称就是*.csdn.net与csdn.net，其中\*是通配符）进行比对，如果域名在"使用者可选域名"中，则认证通过，否则依旧会认证失败。

![使用者可选名称](https://img-blog.csdnimg.cn/20181213000611847.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzMjU2Njg4,size_16,color_FFFFFF,t_70)

3.HTTPS的主要功能？
一个是保证通信加密，另一个是进行身份验证。