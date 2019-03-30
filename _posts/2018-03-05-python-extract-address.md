---
layout: post
title: 使用python提取中文地址描述中的省市区信息
date: 2018-03-05
categories: python
tags: python 我的开源库
---


#  引言
---
在一次建模比赛中，我手头里的原始数据中有一个“地址描述”地段，如下：

| 地址描述        | 
| ------------- |
| 广州国际采购中心1401     |
| 上海市长宁区金钟路658弄5号楼5楼      | 
|徐汇区虹漕路461号58号楼5楼 | 
|济南市历下区和平路34号轻骑院内东二层山东朵拉|

这样的地址描述字段过于随意，很难使用，但是看这些字符串的样子似乎又可以提取出其所在的省、市和区，即使只能够提取出区或者市，如果我们有一个省、市和区的归属数据库的话，应该也能够将剩下的信息映射出来，如果自己写的话肯定很麻烦，还要去网上找数据库，于是我做了一个可以复用的python模块，一条命令就可以将上面的“地址描述”字段转换成如下的样子：


|省              |市              |区              | 
| ------------- | ------------- | ------------- |
|广东省     |广州市  | |
|上海市     |上海市  | 长宁区|
|上海市    |上海市 |徐汇区 | 
|山东省    |济南市 |历下区|

# 模块安装
---
目前支持python3

```c-like
pip install cpca
```

# Github地址
---
更详细的模块介绍见Github上的README
[https://github.com/DQinYuan/chinese_province_city_area_mapper](https://github.com/DQinYuan/chinese_province_city_area_mapper)

如果觉得这个模块对你有帮助的话，请给个star啊

# 基本功能
---
## 分词模式

本模块中最主要的方法是`cpca.transform`，该方法可以输入任意的可迭代类型（如list，pandas的Series类型等），然后将其转换为一个DataFrame，下面演示一个最为简单的使用方法：

```python
location_str = ["徐汇区虹漕路461号58号楼5楼", "泉州市洛江区万安塘西工业区", "朝阳区北苑华贸城"]
import cpca
df = cpca.transform(location_str)
df
```


输出的结果为：

```c-like
   省     市    区          地址
0 上海市 上海市  徐汇区     虹漕路461号58号楼5楼
1 福建省 泉州市  洛江区     万安塘西工业区
2 北京市 北京市  朝阳区     北苑华贸城
```

如果你想获知程序是从字符串的哪个位置提取出省市区名的，可以添加一个`pos_sensitive=True`参数：

```python
location_str = ["徐汇区虹漕路461号58号楼5楼", "泉州市洛江区万安塘西工业区", "朝阳区北苑华贸城"]
import cpca
df = cpca.transform(location_str, pos_sensitive=True)
df
```

输出如下：

```c-like
     省    市    区        地址                  省_pos  市_pos 区_pos
0  上海市  上海市  徐汇区  虹漕路461号58号楼5楼   -1     -1      0
1  福建省  泉州市  洛江区  万安塘西工业区         -1      0      3
2  北京市  北京市  朝阳区  北苑华贸城             -1     -1      0
```

其中`省_pos`，`市_pos`和`区_pos`三列大于-1的部分就代表提取的位置。-1则表明这个字段是靠程序推断出来的，抑或没能提取出来。

默认情况下transform方法的cut参数为True，即采用分词匹配的方式，这种方式速度比较快，但是准确率可能会比较低，如果追求准确率而不追求速度的话，建议将cut设为False（全文模式），具体见下一小节。



## 全文模式

jieba分词并不能百分之百保证分词的正确性，在分词错误的情况下会造成奇怪的结果，比如下面：

```python
location_str = ["浙江省杭州市下城区青云街40号3楼"]
import cpca
df = cpca.transform(location_str)
df
```

输出的结果为：

```c-like
     省    市      区    地址
0  浙江省  杭州市  城区  下城区青云街40号3楼
```

这种诡异的结果是因为jieba本身就将词给分错了，所以我们引入了全文模式，不进行分词，直接全文匹配，使用方法如下:

```python
location_str = ["浙江省杭州市下城区青云街40号3楼"]
import cpca
df = cpca.transform(location_str, cut=False)
df
```

结果如下：

```c-like
   省       市     区         地址
0  浙江省  杭州市  下城区     青云街40号3楼
```

这下就完全正确了，不过全文匹配模式会造成匹配效率低下，
我默认会向前看8个字符(对应transform中的lookahead参数默认值为8)，
这个是比较保守的，因为有的地名会比较长（比如“新疆维吾尔族自治区”），如果你的地址库中都是些短小的省市区名的话，
可以选择将lookahead设置得小一点，比如：

```python
location_str = ["浙江省杭州市下城区青云街40号3楼"]
import cpca
df = cpca.transform(location_str, cut=False, lookahead=3)
df
```

输出结果和上面是一样的。

再举一个例子，这个例子经测试只有使用全文匹配才能匹配出地名，：

 ```python
import cpca
cpca.transform(["11月15日早上9点到11月18日下班前王大猫。在观山湖区"], cut=False, pos_sensitive=True)
 ```
输出为:

```c-like
    省     市      区        地址                                              省_pos 市_pos 区_pos
0  贵州省  贵阳市  观山湖区  11月15日早上9点到11月18日下班前王大猫。在观山湖区     -1     -1     25
```

# 地图绘制
---

模块中还自带一些简单绘图工具，可以在地图上将上面输出的数据以热力图的形式画出来.

这个工具依赖folium，为了减小本模块的体积，所以并不会预装这个依赖，在使用之前请使用`pip install folium ` .

代码如下：
```python
from cpca import drawer
#df为上一段代码输出的df
drawer.draw_locations(df, "df.html")
```
这一段代码运行结束后会在运行代码的当前目录下生成一个df.html文件，用浏览器打开即可看到
绘制好的地图（如果某条数据'省'，'市'或'区'字段有缺，则会忽略该条数据不进行绘制），速度会比较慢，需要耐心等待，绘制的图像如下：

![地图绘制](http://upload-images.jianshu.io/upload_images/10192684-0aa4e302810dd135.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

还有更多的绘图工具请参考[Github上的README](https://github.com/DQinYuan/chinese_province_city_area_mapper)中大标题为“示例与测试用例”的部分。