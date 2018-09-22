---
layout: post
title: spyder3(anaconda3)环境下无法使用mayavi库的解决方案
date: 2018-01-21
categories: python
tags: python spyder mayavi
---
在windows下安装mayavi库会遇到很多困难，网上有很多博文提供了行之有效的解决方案，这里就不赘述了，假设在这里读者已经成功安装好了mayavi库，但是当想在spyder3中使用这个库时却出现错误，网上没搜到解决方案，我便自己折腾了一个。这里的spyder版本是3.1.4，mayavi版本是4.5.0

当按照大神们博客里的方法装好mayavi之后，尝试着输入命令`from tvtk.tools import tvtk_doc`测试一下tvtk库是否安装成功，在cmd命令行和IDLE中都可以正常运行，个人习惯用spyder作为python编辑器，所以紧接着就在spyder的IPython窗口也试了一下这个命令，但是却报了如下错误：
![QtError](http://upload-images.jianshu.io/upload_images/10192684-698110f5e0f1d07e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
 这是什么情况，难道必须投靠pyCharm或者用最原始的IDLE才能继续使用mayavi库吗？出于spyder的忠诚，我决定研究这个问题的解决方法，经过研究，我认为问题的原因可能是spyder3它的用户界面是依赖pyqt5的，当启动spyder时，pyqt5也会跟着启动，并且掩盖掉了之前我们安装的pyQt4，所以在spyder中导入tvtk库时，tvtk库会搜索环境中的pyQt版本，就搜索到了spyder所依赖的那个pyqt5，但是tvtk库又识别不了pyqt5，所以就会报错，我的解决方案如下：
  1. 在spyder的ipython窗口输入`from tvtk.tools import tvtk_doc` ，这个时候会报错，点击错误栈的最后一条，进入这一段源码
  ![错误栈](http://upload-images.jianshu.io/upload_images/10192684-a9b6b5fd62f2497b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  2. 我通过阅读源码发现，这一段源码的功能就是从环境中获取qt版本，按照程序的执行流程，只要把第25行"="右边改成None就可以解决，即改成`qt_api = None`
  ![mayavi源码](http://upload-images.jianshu.io/upload_images/10192684-91c72a39c5d931ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  修改完之后保存，再在spyder的ipython输入`from tvtk.tools import tvtk_doc`，发现可以正常运行了
  ![导入成功](http://upload-images.jianshu.io/upload_images/10192684-dff36b0575c10509.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
   