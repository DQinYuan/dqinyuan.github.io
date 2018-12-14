---
layout: post
title: 大厂常见变态面试题解析（网络方向）（一）从网址输入浏览器到网页呈现在屏幕上的过程
date: 2018-12-14
categories: 面试笔记
tags: 网络 面试
cover: 
---


# 引言
---
在大厂面试过程中经常会遇见一些“变态”的题目，这些题目如果没有提前准备，一时间还真无法下手，这个系列文章我就想总结总结这些经常会被问到的“变态”题目的应对策略。



# 总
---
“说说从网址输入浏览器到网页呈现到屏幕上，这个过程中发生了什么？”



在大厂面试中经常会遇到这个问题，考得就是计算机网络的基础是否扎实，思维是否开放，然而面试过程中尝尝一紧张，就只记得一个HTTP请求了，然后面试官就会很不满意，觉得面试者网络的基础很差，特地写这篇文章总结了一下。



这个过程说起来复杂，但是概括起来说就是以下四个步骤：

- DNS解析
- HTTP请求
- 服务端处理和响应
- 浏览器渲染



# DNS解析
---

首先就是要将域名映射成其对应的IP地址，细节步骤如下：

### 1、去浏览器缓存中寻找映射

浏览器会缓存这种映射关系，如果在浏览器缓存中存在的话就优先在浏览器缓存中查找



### 2、去操作系统缓存中寻找映射

操作系统也会缓存DNS映射，这一步需要设计到系统调用



### 3、去路由器缓存中寻找映射

路由器中同样有缓存



### 4、去本地域名服务器（local name server）中寻找

本地域名服务器（又叫做默认域名服务器）往往就是你在操作系统中配置DNS Server（以win10中DNS配置界面为例）:

![win10中DNS配置界面](https://upload-images.jianshu.io/upload_images/10192684-c893d81a6e433c58.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


不过现在大多数情况下采用的都是自动获取DNS服务器地址了，在你的电脑连上网的一瞬间，电脑会在内网发出一个广播，当本地域名服务器收到这个广播之后会进行响应，然后你的电脑就可以根据这个响应的地址来自动配置DNS Server了。

本地域名服务器会有DNS缓存，如果命中的话就会直接从本地域名服务器中返回，不再继续向上请求。

### 5、继续进行递归查询与迭代查询

如果本地域名服务器中没有命中，那么就必须去更上层的域名服务器中查找了，域名服务器的层级关系如下：
![域名服务器层级关系](https://upload-images.jianshu.io/upload_images/10192684-11f660565ba1ddf4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 - 根域名服务器：最高层次的域名服务器，它不会直接存储域名与ip的映射关系，存储的其实是顶级域名服务器的域名及其ip地址 
 - 顶级域名服务器（TLD服务器）：管理该顶级域名中所有二级域名服务器，当收到查询请求时，可能直接返回结果或者下一步应该查询的域名服务器地址
 - 权限域名服务器：负责一个小区域的域名到ip的映射，当收到查询请求时，可能直接返回结果或者下一步应该查询的域名服务器地址

如果没有命中本地域名服务器的话，就会委托本地域名服务器进行“递归查询”，所谓“递归查询”，就是指本地域名服务器会代替你作为DNS客户端请求根域名服务器。递归查询的含义如图：

![递归查询](https://upload-images.jianshu.io/upload_images/10192684-ee31c3a4c2803245.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



前面介绍过，根域名服务器中并不直接存储域名到ip的映射关系，而是会返回你下一个应该找的域名服务器（某个顶级域名服务器），本地域名服务器在收到下一个应该找的服务器的地址之后会继续去请求下一个服务器，不断重复这个过程，这个过程被称为“迭代查询”：

![迭代查询](https://upload-images.jianshu.io/upload_images/10192684-7d7f8010ae6b4d84.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


请求域名服务器的总体过程如图：

![请求域名服务器的过程](https://upload-images.jianshu.io/upload_images/10192684-597234540287e4fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从图中可见，这个过程是一个递归查询与迭代查询相结合的过程，从主机到本地域名服务器是递归查询，而从本地域名服务器再到更高级别则是迭代查询。

DNS映射只能将一个域名映射成一个IP，但是现在的系统处于性能的需要，往往会将应用拷贝好几份到数台机器上（即有好几个IP），这个时候就需要另外的技术来实现对这些IP的负载均衡：
 - DNS轮询（Round-robin DNS）：对于每个映射请求，顺序选择IP列表的下一个IP返回，循环往复
 - 负载均衡器：一个在特定的IP地址上监听，并负责将请求负载到集群中某台机器的设备。负载均衡器在硬件层面上有f5，操作系统层面上有lvs，软件层面上有Nginx，通过层层负载可以对系统进行垂直扩展，通过和上面DNS轮询配合可以实现系统的水平扩展
 - 地理DNS（Geographic DNS）：根据客户端所处的地理位置，解析到离客户端最近的一台服务器的IP，CDN经常采用这种方式
 - 任播技术（AnyCast）：后面提到的DNS的根域名服务器就是使用的这种技术，不过这种技术因为和TCP协议的适应性不是很好，所以在企业中很少使用。



# HTTP请求
---

解析完成后便会将HTTP报文发送请求过去，HTTP请求是基于TCP连接的，所以会先进行TCP三次握手建立连接，HTTP请求报文的内容主要包括请求行，请求头和请求体构成。

有很多抓取Http报文的工具，比如Chrome浏览器自带的开发者工具，Fiddler等，我用Fiddler随意抓取了一个HTTP报文如下：

```
GET http://pos.baidu.com/qctm?conwid=172&conhei=425 HTTP/1.1
Host: pos.baidu.com
Connection: keep-alive
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 6.2; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/47.0.2526.111 Safari/537.36
Referer: http://img2.mini.cache.wps.cn/mini/static/wps/rightad/42.html
Accept-Encoding: gzip, deflate
Accept-Language: zh_CN
Cookie: pgv_pvi=4605897728; BDRCVFR[feWj1Vr5u3D]=I67x6TjHwwYf0; delPer=0; 
```

可以看出HTTP是一个纯文本的协议，非常地可读，上面报文的第一行`GET http://pos.baidu.com/qctm?conwid=172&conhei=425 HTTP/1.1`，称为请求行，表示使用HTTP的GET方法请求链接` http://pos.baidu.com/qctm?conwid=172&conhei=425`，使用的HTTP协议版本是HTTP/1.1。

第一行以外剩下的内容都是请求头，可以看到请求头携带了很多信息过去，通过User-Agent字段告知了服务器自己的操作系统类型，浏览器类型等等，Referer字段表示这个请求来源于哪个页面，自己接受什么类型的响应（Accept-Encoding和Accept-Language）等等，w3c有一套完整的规范（RFC文档）约束HTTP请求头中有哪些字段以及每个字段的含义。

HTTP协议是一个无状态的协议，即服务器无法感知到两次请求的不同，为了弥补这个不足，这个请求头还携带了Cookie，Cookie本质上就是一个用于在不同请求之间同步用户状态的kv对，这个Cookier是上一次请求服务器时服务器响应给我的，浏览器会负责维护属于不同网站的Cookie，在请求需要时自动将其携带进请求头中。

在该请求头中还有一个`Connection: keep-alive`字段，这个实在HTTP/1.1中新增的字段，之前的HTTP/1.0，默认每次HTTP连接都是短连接，即完成一个响应后就立即关闭，使用了keep-alive就可以多个请求响应共用一个tcp连接，节约创建和销毁链接的开销。

该请求报文中没有请求体，原因是GET方法的HTTP请求都不包含请求体，从字面意思可以看出"GET"是获取内容的意思，仅仅是获取内容当然就不需要自己携带什么参数，所以就没有请求体，当然实际开发中如果实在想要通过GET方法传参数的话，可以通过在请求的URL后面加个"?"以及kv对的形式，比如之前那个请求url：`http://pos.baidu.com/qctm?conwid=172&conhei=425`，可以改看出它请求`http://pos.baidu.com/qctm`，并且传递了两个参数，一个是`conwid`，值为`172`，另一个是`conhei`，值为`425`。

除了GET方法外，开发中最经常用的就是POST方法，POST方法是可以携带请求体的，只需要把上面所说的kv对放入请求体中即可达到和上面一样的携带参数的效果（请求体和请求头中间要空一行），使用POST方法的优势就是，当报文内容太多时，超出了url的最大长度限制，GET方法无法承受，此时最好就使用POST方法了。

GET方法和POST方法还有一点重大区别就是，GET方法会优先使用浏览器缓存，而POST方法每次都会重新请求，很多情况下选择使用GET方法就是为了合理使用浏览器缓存来提升用户体验。

GET方法和POST方法外加携带一些参数理论上已经可以满足全部的开发需求，但是HTTP远不止这两种方法，HTTP的方法与含义如下：
    ![HTTP的方法](https://upload-images.jianshu.io/upload_images/10192684-6f99bcc6f5c7f3c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

近些年兴起的RESTful风格的url主张要充分利用HTTP中这些方法的语义，尽可能地将参数放置在url路径中，而不是总是使用GET或者POST方法传一堆杂乱的参数，一般认为RESTful风格的URL更加可读。

上面说过HTTP协议是一个纯文本协议，因而保密性不是很好，而且也缺乏身份认证机制，所以现在现在大多数网站用的都是http的一个改良版本，称为HTTPS，它在HTTP和TCP之间增加了一个SSL层（安全套接字层），它占用的是443号端口，而HTTP占用的是80号端口，现在的趋势是全网使用HTTPS，以保证网络传输的安全，使用fiddler想抓到纯http的包很难了，想要更多地了解HTTPS的话可以参考我之前的写的一篇文章：[HTTPS原理总结](https://www.dqyuan.top/2018/01/24/https-summary.html)


# 服务端处理和响应
---
按照现代动态网站后端的一般架构，在服务端会先进行好几轮的负载均衡，然后才能到达真正处理请求的后端程序，如下图：
![lvs和nginx负载均衡](https://upload-images.jianshu.io/upload_images/10192684-d2a9cb239c726fe1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这之前在将DNS对IP的负载时也提到过一些，你可能会奇怪为什么图中第一层的负载均衡器lvs有两个，其实在同一个时刻，只有其中一个lvs在起作用，它叫做Master，另一个叫做Backup，当Master因为意外而退出的时候，此时Backup就会切换为Master，负责这个监测和切换工作的就是图中的Keepalived软件，它使用的技术称为虚拟ip(virtual ip)，在一开始的时候Master占有这个虚拟ip，Master挂了后，Backup则会将这个虚拟IP抢占过来。

经过了负载均衡后达到了后台应用，后台应用就是一段用于处理请求的程序（可能是由Java，Python，Ruby，Scala等各种编程语言编写），这些语言一般也都有自己的应用容器，其实也是个服务器程序，比如Java里面最经常使用的tomcat，程序所做的事情大多就是对数据库进行各种增删改查，企业中最常用的数据库就是mysql和orcale，当网站的数据量或者访问量特别大的时候，可能需要考虑进行分库分表，方法包括分片（Sharding），复制（Replication），或者使用一些实现了弱一致性语义的数据库，比如MongoDB，HBase等等。

后端程序处理完后会返回给客户端一个HTTP响应，HTTP响应的格式与HTTP请求的格式类似：

```
HTTP/1.1 200 OK
Cache-Control: private, no-store, no-cache, must-revalidate, post-check=0,
    pre-check=0
Expires: Sat, 01 Jan 2000 00:00:00 GMT
P3P: CP="DSP LAW"
Pragma: no-cache
Content-Encoding: gzip
Content-Type: text/html; charset=utf-8
X-Cnection: close
Transfer-Encoding: chunked
Date: Fri, 12 Feb 2010 09:05:55 GMT

2b3
��������T�n�@����
```

第一行是响应行，再往下面的内容，空行之前是响应头，空行之后是响应体。

乱码部分是经过压缩的响应体，压缩算法就是响应头中`Content-Encoding`字段中的`gzip`算法，进过gzip解压缩即可看到实际内容：

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN"   
      "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en" 
      lang="en" id="facebook" class=" no_js">
<head>
<meta http-equiv="Content-type" content="text/html; charset=utf-8" />
<meta http-equiv="Content-language" content="en" />
...
```
往往会发现其实是一个html页面。

响应行由协议版本，响应码和响应码的描述组成，200就是代表成功的响应码，还有其他很多其他响应码，但是他们一般遵循以下规律：

| 响应码        | 含义    |
| :--------   | :-----   |
| 1xx       |表示通知信息，如请求收到了或者正在处理  | 
| 2xx        | 表示成功      |
| 3xx        | 表示重定向，如要完成请求还必须进一步的行动      | 
| 4xx            | 客户端错误           |
|5xx         | 服务器错误 |


# 浏览器渲染页面
---

浏览器通过上面的请求拿到html后就开始进行渲染，浏览器会先渲染出一个大概的结构出来，因为现在前端各项技术的分工非常明确，通过html浏览器只能知道结构，然后去请求嵌入在html中的实体，比如图片，css和js，通过css浏览器就知道了网站的样式，通过js浏览器就知道了网站的行为

 - 图片: img标签
```html
<img src="https://xxxxxx/xxxx.png"/>
```
 - css样式表，用于决定网站的样式
```html
<link href="https://xxxxx/bootstrap/3.3.4/css/bootstrap.min.css" rel="stylesheet">
```

 - js, 即JavaScript，用于决定网站的行为  

```
 <script src="http://xxxxxxx/jquery/2.1.4/jquery.min.js"></script>
```

当遇到上面三种标签的时候，浏览器都要为他们每一个单独发起一个HTTP请求，这中间的过程和前面说的过程一样。

但是对于html，css和js这样的静态资源，基本上是走不到网站的后端的，首先浏览器会缓存他们，通过响应头的`Expires`字段，浏览器知道他们什么时候会过期，如果没有过期，就直接从浏览器缓存中获取了。还有，返回html页面的那个响应报文上很有可能会有一个`ETag`字段，`ETag`类似于版本号，如果浏览器发现自己的缓存里有这个版本号的html的嵌入实体资源，则直接从缓存返回。

即使浏览器缓存中没有，大多数现代的网站都把静态资源托管到CDN上了，CDN全称叫内容分发网络，CDN服务器遍布全国各地，你对于静态资源的请求往往会被负载到一个比较近的CDN服务器上。

在html中的各项嵌入实体也加载完毕后，浏览器开始发出AJAX请求。

在过去AJAX技术没有被广泛使用的时候，每次页面有一丁点动态变化（哪怕只改了一个字），都必须向后端请求一个完整的新网页，这显然是很低效的，利用这项技术可以每次只向后端索要少量的数据然后只刷新部分网页，而不用每次都去请求一个完整的网页。

上面的介绍听上去比较玄乎，其实就是在网页的js代码中发起http请求向服务器要数据，数据来了后再使用js代码根据数据对页面进行动态更新，从这里可以看出前端变得比以前复杂了。

AJAX这个看上去很诡异的单词其实是"Asynchronous Javascript And XML"的缩写，中文可以强行翻译为"异步Javascript和XML"，然而名不副实，现在AJAX的前后端交互基本上传输的都是json格式，xml格式已经用得非常少了，但是这个名称还是遗留了下来。

AJAX技术催生了一种叫做WebAPP的架构，举个例子，现在有很多网页版office，网页版photoshop，以及种种将复杂交互集中于一个页面的应用，这些应用往往只需要在你第一次访问的时候将页面下载到本地，之后都使用AJAX与后端交互，然后就能给人一种很接近本地应用的感觉。

AJAX也带来了前后端分离的趋势，以前前端只负责写好一个静态的页面，然后后端负责使用模板语言为其添加动态效果，这个模板常常会导致一个分工不明确的模糊地带，前后端分离则主张将后台彻底变成一个数据接口，所有交互都由前端掌控，听上去就是WebAPP的架构，然而并不是所有应用都是WebAPP，淘宝网首先想到了在前端和后端之间引入一个NodeJS中间层（其实NodeJs就是一个能够执行js语言的服务端应用容器），将前端掌控的范围进一步扩大到了服务端，更精确的说是MVC中的Controller，这样方便前端进一步掌控交互逻辑，对交互的性能进行优化。

如果页面渲染时间太长，会给用户不太好的体验，现在有很多降低渲染时长的方法，比如FaceBook发明的BigPipe技术，正常情况下用户体验到的动态网站的延时由三部分组成：服务器生成页面的时间，网络传输的时间和浏览器渲染的时间

![网站延时的组成](https://upload-images.jianshu.io/upload_images/10192684-aa31cda99dab2136.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而BigPipe则将页面分为很多个部分(称为一个pagelet)，然后在服务器和浏览器之间建立一个持续的连接，每一部分类似于流水线一样源源不断地从服务器传输到浏览器。BigPipe首先会选择输送一个框架性的HTML结构，然后浏览器就可以先根据这个结构进行渲染，然后其他部分再源源不断地加载进来，用户就会感觉变快了：

![BigPipe](https://upload-images.jianshu.io/upload_images/10192684-c583a7f69edfd4b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

BigPipe的实现依赖于HTTP 1.1所引入的分块传输编码，即HTTP的头部字段`Transfer-Encoding`的值为`chunked`，那么消息体就可以由数量不确定的块组成，并且以最后一个大小为0的块结束。

# 其他补充
---

从上面的流程来看，都是浏览器在需要资源的时候，主动发送请求去服务器上拉（pull）数据，但是有的场景下是需要服务器主动向客户端推（push）数据的，比如网页版的聊天室，当你的朋友向你发送一条消息，服务器收到后，必须主动将消息推给你，你才能在浏览器上看到。

然而HTTP原生是不支持服务器"推送"的，所以人们在一开始发明很多投机取巧的方法，称之为Comet，Comet有两种经典的实现，一个是长轮询（long polling），即客户端发出AJAX请求后，服务器将请求一直阻塞在那里直到超时或者服务器有数据推送，一条连接超时后，浏览器紧接着开启下一个长连接，因为每次连接的时间都很长，所以称之为“长轮询”。另一个种实现就是借助iframe标签的src属性，然后服务器就像“长轮询”的那样把iframe标签的请求阻塞住，推送，超时，重建请求。

后来浏览器开始支持WebSocket协议，这是一个双向的通道，服务端和浏览器只需要进过一次HTTP请求进行协议升级后，即可进行基于WebSocket的双向通信。

在HTTP/2中直接将服务器推送加入HTTP协议中。

# End
---

如果面试中能提到上面总结的这些要点，面试官应该会比较满意。


# 参考文献
---

 1.  国外大佬的博客： http://igoro.com/archive/what-really-happens-when-you-navigate-to-a-url/
 2.  《计算机网络》（第五版，谢希仁著）
 3.  ETAG：https://www.cnblogs.com/softidea/p/5986339.html
 4.  Long Polling：https://www.jianshu.com/p/d3f66b1eb748?from=timeline&isappinstalled=0
 5.  BigPipe：http://taobaofed.org/blog/2015/12/17/seller-bigpipe/
 6.  负载均衡：https://www.cnblogs.com/arjenlee/p/9262737.html
 7. 《Web全栈工程师的自我修养》（余果 著）
 8. Comet: https://www.ibm.com/developerworks/cn/web/wa-lo-comet/