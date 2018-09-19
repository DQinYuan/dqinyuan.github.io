---
layout: post
title: 使用spyder3调试python程序的简明教程
date: 2018-01-21
categories: python
tags: python spyder debug
---

说是简明教程，其实是我自己尝试用spyder调试python程序的过程的一个记录，因为spyder的调试功能是基于pdb，而我又没有pdb的基础，所以刚开始上手时感觉很不习惯，而且那时我又很懒，没去找官方文档，仅仅在百度和csdn上找了找，没找到比较好的资料，于是放弃了，过了一段时间之后，突然又心血来潮去找了官方文档，外加自己的一些尝试，总算入门了spyder的调试功能，特地记录下来与大家共享,我使用的spyder版本是3.1.4(使用pip list命令查看spyder版本)

# Spyder官方文档地址
---
http://pythonhosted.org/spyder/

# 开始调试
---
先写一个简单的小程序用于调试：
```python
# -*- coding: utf-8 -*-
"""
Created on Mon Aug 28 23:59:40 2017

@author: 燃烧杯
"""

a = 'a'
b = 'b'
c = 'c'
e = 'e'
f = 'f'
g = 'g'
h = 'h'
print(a)
```
我们暂时先不打断点，用debug的方式运行该代码试试
![debug](http://upload-images.jianshu.io/upload_images/10192684-b17974149c7b1105.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

点击spyder工具栏上的Debug file按钮，或者使用快捷键Ctrl+F5开始调试。

在ipython界面会输出如图所示的内容：
![first debug](http://upload-images.jianshu.io/upload_images/10192684-7a838ea77c48d093.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

出现了ipdb提示符，说明我们已经进入了调试模式，上面输出的内容可以看出是代码的第一行，接着在提示符中输入c(continue的缩写，表示程序继续向下执行到下一个断点)，会输出如下内容：
![first_debug_end](http://upload-images.jianshu.io/upload_images/10192684-b2c89d33f720bec2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

程序执行结束，可见即使我们没有打断点，仍然会在第一句执行之前中断（经测试，中断的时候第一句还没有执行），这个和我用过的其他编译器不太一样（比如eclipse和IntelliJ，在不打断点的情况下会正常执行到底），一开始还让我困惑了一下，后来就适应了.
如果你仔细看刚才的工具栏截图的话，会发现在debug按钮组的第五个按钮和刚刚的'c'命令是一样的功能，但是不知道为什么，在我这个版本的spyder里有这个按钮一些bug（具体来说就是在程序执行结束之后不会自动退出pdb，而且之后再想使用'q'命令退出也退出不了，换而言之，就是卡死在了pdb里面），如果你使用的是更高版本的spyder的话，这个bug可能已经修复了，可以尝试一下.

# 打断点的两种姿势
---
## 普通的breakpoint
用spyder打断点的方法非常简单，只要在想打断点的那一行行首双击鼠标即可，如图所示，我们尝试建立一个断点：
![break_point](http://upload-images.jianshu.io/upload_images/10192684-621a3d0e4b7dc628.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在每次开始debug之前，先在spyder的ipython界面中输入`%reset` 把工作空间的所有变量清除，以免影响到我们接下来的测试.
按下Ctrl+F5开始debug，进行如图所示的操作：
![to_breakpoint](http://upload-images.jianshu.io/upload_images/10192684-8d562d52dede324f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后我们就到达了断点处，从箭头(-->)以及`d:\ide\pyproject\pdbtest\test1.py(12)<module>()` 中的数字12可以看出程序刚刚执行到了第12行（也就是我们打断点的这一行），第12行到底有没有执行呢？只要测试一下f变量是否存在就可以了，尝试在ipdb中进行如下输入：
![ipdb](http://upload-images.jianshu.io/upload_images/10192684-21f332d67fb63896.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`!(python语句)`的意思就是在当前状态下执行该python语句，我刚刚的用法的意思显然是查看变量内容，从`!f` 的错误信息可以看出f尚未定义，即第12行代码(`f='f'`)还没有执行，查看e变量发现e变量已经被定义了，这说明第11行已经执行结束了。通过以上实验可以看出，spyder会在断点语句的执行之前中断

## 带条件的breakpoint
双击刚刚在第12行代码开头创建的“小红点”即可取消断点。
按住Ctrl+Shift，然后像刚才一样双击第12行行首，会弹出一个小框：
![condition](http://upload-images.jianshu.io/upload_images/10192684-c36fc6f1dd8d1fe3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这个小框内可以输入断点的条件，可以是任意返回True或False的python语句，比如我输入
```python
(a==4)and(b==5)
```
然后点击OK按钮，发现小红点上多了一个问号，这个表示条件断点(conditional breakpoint)，开始debug试一下.
![debug](http://upload-images.jianshu.io/upload_images/10192684-ae1299ac4401a883.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
发现程序只在第一句中断一下，断点根本就没有起作用，因为在断点的时候，a变量为'a'，b变量为'b'，不符合条件当然不会中断.

现在重新开始debug，然后连续按三遍Ctrl+F10，然后发现程序执行到了第十行：
![ctrl_f10](http://upload-images.jianshu.io/upload_images/10192684-63df4b9cd56aa05f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其实Ctrl+F10是单行执行的意思，每按一次执行一行，相当于点击了工具栏上如下图所示的按钮：
![run_current_line](http://upload-images.jianshu.io/upload_images/10192684-8ee5ec5f82beddf2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个时候我可以使用刚刚讲过的`!(python语句)`来给a,b临时指定一个值，在ipdb的提示符中输入`!a=4;b=5` ，然后使用c命令继续执行，发现在条件断点处中断了，因为此时满足了我们刚刚给条件断点指定的条件：
![condition_break](http://upload-images.jianshu.io/upload_images/10192684-8b254485d65db5f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果忘记了条件断点的条件是什么的话，可以按住Ctrl+Shift键双击“带问号的小红点”，然后就能看见条件是什么了，而且还可以修改条件，如果要取消断点的话，直接双击就可以了。

# 总结一下刚刚所讲的
---
 - `Ctrl+F5` 以Debug模式运行文件
 - 在debug之前记得用`%reset` 指令清空一下ipython工作空间中的变量，以免影响debug中变量值的查看
 - 无论你是否打断点，都会在第一行语句执行之前中断一次
 - !(python语句)可以在pdb提示符下执行python语句，可以用来查看变量值或者给变量临时指定值
 - c命令或者`Ctrl+F12`可以让程序执行到下一个断点
 - q命令退出调试
 - `Ctrl+F10` 单行执行
 - 双击行首设置断点，按住`Ctrl+Shift` 双击行首可以设置条件断点

# 剩下的一些细节
---
上面的例子已经包括了大多数常用的功能，如果曾经用过别的编译器的调试功能的话（如eclipse和IntelliJ等），看到这里就可以了，对于有调试经验的人来说，我下面要讲的两个功能只要看到按钮的名称就大概知道它是做什么的了.
如下：
![step_into](http://upload-images.jianshu.io/upload_images/10192684-90b2f7174e52d20e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![return](http://upload-images.jianshu.io/upload_images/10192684-07b6942e912c3814.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Step Into
上面一张图的按钮的功能我们称之为Step Into(下面一张图的按钮的功能我们称之为return)，用于进入一个函数体内部，为了更清楚的说明它的功能，我们给出一个例子，在spyder中创建如下程序：
```python
# -*- coding: utf-8 -*-
"""
Created on Tue Aug 29 14:22:46 2017

@author: 燃烧杯
"""

def myTest():
    c = 'a'
    d = 'b'
    e = 'c'
    return c

a = 'a'
b = 'b'
c = myTest()
f = 'f'
print(a)
```
我们开始debug，不断地按`Ctrl+F10` 单行执行这个程序，当运行到`c = myTest()` 这句时注意一下：
![not_step_into](http://upload-images.jianshu.io/upload_images/10192684-667be2d82e3e2d86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

不管myTest()中有多少代码都直接当做一行跳了过去，用q命令退出调试。
重新debug该文件，单行执行到`c = myTest()` 这行时按`Ctrl+F11` 使用Step Into功能，发现我们进入了函数内部的代码段：
![step_into](http://upload-images.jianshu.io/upload_images/10192684-2e061f2a276b3312.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这就Step Into的主要功能.

## Return
Return的功能与Step Into的功能刚好相反，当使用Step Into进入函数之后，按`Ctrl+Shift+F11` 后会直接跳到该函数的执行的最后一行，此时在按一遍`Ctrl+Shift+F11` 或者`Ctrl+F10` （单行执行）就可以跳出函数了，想要尝试的话可以自行在我上面给出的例子中尝试.

# End
---
感谢阅读，希望世界上的bug越来越少（手动滑稽）

