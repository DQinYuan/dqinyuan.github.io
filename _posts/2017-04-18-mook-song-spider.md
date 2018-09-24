---
layout: post
title: 慕课BIT嵩老师爬虫课笔记
date: 2017-04-18
categories: python
tags: python 爬虫
---

一直想学python，在中国大学慕课上发现了嵩老师的python系列专题课程，就首先试着听了听嵩老师的爬虫课，虽然时间有点短并且内容也不够深入，但是讲得很生动，于是就记了一些笔记，也稍微加入了一些自己的见解和总结

# Requests库
---
## 主要方法

 - r = requests.get(url)
 - r = requests.get(url, params=None, **kwargs),params指url中额外的访问参数，kwargs指12个控制访问参数
 - r = requests.post(url, data=None, **kwargs)
 - r = requests.request('get', url, params, **kwargs)
 - r = requests.head(url)    只获得请求头
 >一些常用的访问控制参数：
 >params    请求参数
 >data         请求体参数
 >json          json格式数据作为请求体内容
 >headers     定制请求头
 >cookies      字典或者CookieJar
 >auth          元组，支持http认证功能
 >files           字典类型，传输文件
 >proxies      字典，设置访问代理服务器
 >allow_redirects   重定向开关，默认开
 >verify         认证ssl开关，默认开
 >cert            本地ssl路径
 
## Response对象的常用属性
 - r.status_code     返回状态码
 - r.text                   响应内容的字符串形式，即url对应的页面，如果不是get的请求方式的话则打印的是请求体
 - r.encoding         header中的响应编码方式
 - r.apparent_encoding  从内容中分析出的响应编码方式
 - r.content           响应内容的二进制形式
 - r.url                   请求时的url

>1.常用r.encoding = r.apparent_encoding来防止页面乱码。
>
>2.使用r.raise_for_status()在返回状态不是200时产生异常


## 其他使用技巧
### 伪装成浏览器：
```python
r=requests.get(url,headers={'user-agent':'Mozilla/5.0'})
```
      

# Robots协议
---

## 查看方法
url/robots.txt


# BeautifulSoup库

---
## 获得soup
```python
soup=BeautifulSoup('<html>...</html>','html.parser')
```
常用解析器：

 - **html.parser** bs库自带的解析器，解析html;
 - **lxm**  需要安装lxml库，解析html,xml;
 - **xml**   需要安装lxml库，解析xml，笔者使用时感觉对xml文档的容错能力较低，不如lxml;
 - **html5lib** 需要安装html5lib
  
## BeautifulSoup基本元素
 - **Tag** 标签
 - **Name** 标签名，可通过tag.name获得
 - **Attributes** 标签属性，字典形式组织，通过tag.attrs获得
 - **NavigableString** 非属性字符串，可通过tag.string获得
 >注：如果标签内有多个分离的字符串，则返回None，用tag.text则可以返回由这些分离的字符串连接起来的str类型
 
 - **Comment** 标签内注释
 

## 标签树的遍历
### 下行遍历

 - **.contents** 返回直接子节点的列表
 - **.children**  返回直接子节点的迭代类型
 - **.descendants** 返回所有后裔节点的迭代类型
### 上行遍历

-  **.parent** 节点的父标签
- **.parents** 节点的所有先辈标签的迭代类型

### 平行遍历

 - **.next_sibling** 下一个平行节点标签
 - **.next_siblings** 返回所有后续平行节点的迭代类型
 - **.previous_sibling** 上一个平行节点标签
 - **.previous_siblings** 返回前续所有平行节点的迭代类型
 > 注：平行遍历发生在同一父节点下
 
 
## 友好显示
```python
print(soup.prettify())
```

## 常用的信息表示形式与信息提取方法

 - 形式：xml,json,yaml
 - 提取方法：形式解析与搜索方法

## BeautifulSoup标签查找方法
- **tag.find_all(name,attrs,recursive,string,\**kwargs)** 返回所有符合要求的标签列表，name为要检索的标签名；attrs限定属性；recursive表示是否递归检索所以子孙，默认true；string表示对字符串区域检索字符串，如果限定了该属性，此时返回的是字字符串列表。
>注：tag('a')等价与tag.findall('a')
- **tag.find()** 返回搜索到的第一个结果
 > 注：tag.a等价于tag.find('a')

- **tag.find_parents()** 在所有先辈节点中搜索，返回一个列表返回所有查找到的结果
- **tag.find_parent()** 在所有先辈节点中返回第一个匹配结果
-  **tag.find_next_siblings()** 在后续平行节点中返回一个列表包含所有所有匹配结果
-  **tag.find_next_sibling()** 返回后续节点中第一个匹配的结果
-  **find_previous_siblings()**
- **find_previous_sibling()**
>注：所有这些函数的参数都同find_all()

## 案例：中国大学排名定向爬虫

### format函数
```python
"{:^10}\t{:^6}\t{:^10}".format("排名","学校名称"，"总分")
```
解决中文对齐问题：
```python
"{0:^10}\t{1:{3}^6}\t{2:^10}".format("排名","学校名称","总分",chr(12288))
```
>chr(12288) 表示中文空格:{3}表示用format函数中索引为3的参数进行填充

# Re库
---
## 常用符号

`.`  | 匹配任意单个字符
`[]` | 字符集，对单个字符给出取值范围
`[^]`|  “非”字符集，对单个字符给出排除范围
`*` | 前一个字符0次或多次扩展
`+`  | 1次或多次扩展
`?`  |  0次或1次扩展
`|` |   左右表达式任意一个
`{m}`|  前一个字符扩展m次
`{m,n}`|  扩展m至n次(含n)
`^`   | 匹配字符串开头
`$`  | 匹配字符串结尾
`()`  |  
`\d`  | 数字，等价与[0-9]
`\w`  | 单词字符，等价与[A-Za-z0-9_]

## 经典正则表达式

`^[A-Za-z]+$`  | 由26个字母组成的字符串
`[\u4e00-\u9fa5]` |  匹配中文字符
`(([1-9]?\d|1\d{2}|2[0-4]\d|25[0-5]).){3}([1-9]?\d|1\d{2}|2[0-4]\d|25[0-5])` |   匹配ip地址
 
 
## Python的raw string类型
 `r'text'`   则python不会对字符串中的转义符进行转义，正则表达式一般使用raw string

## Re库主要函数

 - **re.search(pattern,string,flags=0)**  搜索匹配正则表达式的第一个位置，返回match对象
 - **re.match()**  从字符串开始位置匹配正则表达式，返回match对象，可以使用在if语句中
 - **re.findall()**  以列表类型返回全部能匹配的子串
 - **re.split(pattern,string,maxsplit=0,flags=0)**    按照正则表达式匹配结果对字符串进行分割，返回列表类型;maxsplit表示最大分割数，剩余部分作为最后一个元素输出
 - **re.finditer()**    返回匹配结果的迭代类型，每一个迭代元素是match对象
 - **re.sub(pattern,repl,string,count=0,flags=0)**    替换所有匹配正则表达式的子串，返回替换后的字符串;repl表示替换匹配字符串的字符串，count表示最大替换次数

>flags包括如下：
 - **re.IGNORECASE  re.I**  忽略大小写匹配
 - **re.MULTILINE  re.M**     正则表达式的^操作符能将每行当做匹配开始
 - **re.DOTALL  re.S**       正则表达式的.操作符可以匹配包括换行符在内的一切字符，如果不指定这个模式，则.不能匹配换行符
 
## Re库的另一种等价用法
```python
pat=re.compile(r'[1-9]\d{5}')
rst=pat.search('BIT 100081')
```

## Match对象介绍
Match表示一次匹配的结果，包含很多匹配信息
### Match对象的属性

 - **.string**    待匹配文本
 - **.re**    匹配时使用的pattern对象
 - **.pos**   正则表达式搜索文本的开始位置
 - **.endpos**   正则表达式搜索文本的结束位置
 
### Match对象的方法
 - **.group()**   获得匹配后的字符串
 - **.start()**    获得匹配字符串在原始字符串的开始位置
 - **.end()**    匹配字符串在原始字符串的结束位置
 - **.span()**    返回(.start(),.end())


## 贪婪匹配与最小匹配
Re库默认采用贪婪匹配

### 最小匹配操作符

 - *?
 - +?
 - ??
 - {m,n}?

## 案例
网站选取心态：不要纠结与某个网站，多找信息源尝试

判断一个元素是否是标签：
```python
import bs4
isinstance(tr, bs4.element.Tag)
```
>注：前面说的那几个BeautifuSoup元素都是在bs4的element的模块下

# Scrapy框架入门
---
## 框架结构
![Scrapy的框架结构](http://ou427u8j5.bkt.clouddn.com/scrapy%E6%A1%86%E6%9E%B6%E7%BB%93%E6%9E%84.png)
5个组件，2个Middleware，包括Downloader Middleware和Spider Middleware

## Scrapy常用命令行

 - **startproject** 创建一个新工程
 - **genspider** 创建一个爬虫
 - **settings**  获得爬虫配置信息
 - **crawl**    运行一个爬虫
 - **list**    列出工程中所有爬虫
 - **shell**   启动url调试命令行

## Scrapy爬虫编写步骤
### 步骤1
建立一个scrapy爬虫工程，即生成工程目录
```c-like
scrapy startproject 工程名
```

### 步骤二
在工程中产生一个scrapy爬虫。

进入刚刚创建的目录，输入如下命令：
```c-like
scrapy genspider 爬虫名称 初始url
```
spiders目录下会生成 爬虫名称.py，该命令仅用于生成 爬虫名称.py，该文件也可以手动生成。


### 步骤三

 -  配置初始url：
     打开刚刚生成的py文件，修改start_urls变量
  
 -  编写解析方式：
     修改parse函数

示例如下：
![代码示例](http://ou427u8j5.bkt.clouddn.com/scrapy%E4%BB%A3%E7%A0%81%E7%A4%BA%E4%BE%8B.png)
另一个等价版本，使用yield生成器：
![yield版本代码示例](http://ou427u8j5.bkt.clouddn.com/scrapy%E4%BB%A3%E7%A0%81%E7%A4%BA%E4%BE%8B%E7%9A%84yield%E7%89%88%E6%9C%AC.png)

### 步骤四
运行爬虫，命令如下：
```c-like
scrapy crawl 爬虫名称
```

## yield关键字
包含yield语句的函数是一个生成器，生成器每次产生一个值，然后被冻结，被唤醒后再产生一个值，不断往复。

示例：
```python
def gen(n):
    for i in range(n):
        yield i**2

for i in gen(5):
    print(i, " ", end="")
```
输出：
```c-like
0 1 4 9 16
```

## Scrapy爬虫框架的使用步骤

 1. 创建一个工程和Spider模板
 2. 编写Spider
 3. 编写Item Pipeline
 4. 优化配置策略

## Scrapy中的几种数据类型
### Request类
含义：表示一个http请求，由Spider生成，Downloader执行

属性和方法：

 - **url**   请求的url地址
 - **method**    请求对应的http方法，'get','post'等
 - **headers**    字典类型风格的请求头
 - **body**    请求主体内容，字符串类型
 - **meta**     用户添加的拓展信息，在Scrapy内部模块间传递信息使用
 - **.copy()**  复制该请求

### Response类
含义：表示一个http响应，由Downloader生成，Spider处理

属性和方法：

 - **url**    对应的url地址
 - **status**    响应状态码
 - **headers**
 - **body**
 - **flags**    一组标记
 - **request**    产生Response类型所对应的Request对象
 - **copy()**    复制该响应

### Item类
含义：表示一个从html页面中提取的信息内容，由Spider生成，Item Pipline处理，Item类似字典类型，可以按照字典类型操作

## Scrapy支持的信息提取方法

 - BeautifulSoup
 - lxml
 - re
 - XPath Selector
 - CSS Selector
>CSS Selector示例：
>    response.css('a::attr(href)').extract()
> 表示抽取a元素的href属性，返回一个迭代类型

## 股票数据爬虫案例
### 非终结parse方法
是一个Request对象的生成器
```python
def parse(self, response):
    for href in response.css('a::attr(href)').extract():
        try:
            stock = re.findall(r"[s][hz]\d{6}", href)[0]
            url = 'https://gupiao.baidu.com/stock/' + stock + '.html'
            yield scrapy.Request(url, callback=self.parse_stock)
        except:
            continue
```
### 终结parse方法
也是一个生成器
```python
def parse_stock(self, response):
    infoDict = {}
    stockInfo = response.css('.stock-bets')
    name = stockInfo.css('.bets-name').extract()[0]
    keyList = stockInfo.css('dt').extract()
    valueList = stockInfo.css('dd').extract()
    for i in range(len(keyList)):
        key = re.findall(r'>.*</dt>', keyList[i])[0][1:-5]
        try:
            val = re.findall(r'\d+\.?.*</dd>', valueList[i])[0][0:-5]
        except:
            val = '--'
        infoDict[key]=val
    infoDict.update(
        {'股票名称': re.findall('\s.*\(',name)[0].split()[0] + \
         re.findall('\>.*\<', name)[0][1:-1]})
    yield infoDict
```
### 编写Pipelines
步骤：
 1. 配置pipelines.py
 2. 定义对爬取项的处理类
 3. 配置settings.py的ITEM_PIPELINES选项
 示例：

编写pipelines.py:

方法运行顺序为open_spider->process_item->close_spider
```python
class BaidustocksInfoPipeline(object):
    def open_spider(self, spider):
        self.f = open('BaiduStockInfo.txt', 'w')
 
    def close_spider(self, spider):
        self.f.close()
 
    def process_item(self, item, spider):
        try:
            line = str(dict(item)) + '\n'
            self.f.write(line)
        except:
            pass
        return item
```

配置settings.py:
```python
ITEM_PIPELINES = {
    'BaiduStocks.pipelines.BaidustocksInfoPipeline': 300,
}
```
300的大小表示执行顺序
### 优化配置

 - **CONCURRENT_REQUESTS** Downloader最大并发下载数量，默认32
 - **CONCURRENT_ITEMS**    Item Pipeline最大并发ITEM处理数量，默认100
 - **CONCURRENT_REQUESTS_PER_DOMAIN**    每个目标域名最大的并发请求数量，默认8
 - **CONCURRENT_REQUESTS_PER_IP**    每个目标IP最大并发请求数量，默认0，非0有效
