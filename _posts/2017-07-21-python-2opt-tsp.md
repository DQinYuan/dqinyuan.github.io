---
layout: post
title: 2-opt求解TSP（旅行商）问题的python实现
date: 2017-07-21
categories: python
tags: python tsp 算法
---

2-opt其实是2-optimization的缩写，简言之就是两元素优化。也可以称作2-exchange 。（摘自百度百科）

这个一种随机性算法，基本思想就是随机取两个元素进行优化，一直到无法优化为止。在小规模TSP问题上，2-opt无论从效率还是效果上都优于蚁群算法。

最初这个算法就是在解决TSP问题上取得了比较好的成效，这里也以TSP问题为例。

**TSP（旅行商）问题**：假设有十一座城市，一位旅行商要经过这十一座城市并且最终返回出发的城市，求解最短的路线。

使用2-opt思想解决该问题的算法如下（首先设置参数最大迭代次数maxCount，初始化计数器count为0）：
 

 1. 随机选择一条路线（比方说是A->B->C->D->E->F->G），假设是最短路线min；
 2. 随机选择在路线s中不相连两个节点，将两个节点之间的路径翻转过来获得新路径，比方我们随机选中了B节点和E节点，则新路径为A->(E->D->C->B)->F->G，()部分为被翻转的路径;
 3. 如果新路径比min路径短，则设新路径为最短路径min，将计数器count置为0，返回步骤2，否则将计数器count加1，当count大于等于maxCount时，算法结束，此时min即为最短路径，否则返回步骤2;
 
 在网上没能找到相关的python代码实现，于是我就自己粗糙地实现了一个（数据来源于某次华为杯数学建模竞赛）:
 
```python
# -*- coding: utf-8 -*-
"""
Created on Fri Jul 21 13:47:57 2017

@author: 燃烧杯
"""

import numpy as np
import matplotlib.pyplot as plt

#在这里设置迭代停止条件，要多尝试一些不同数值，最好设置大一点
MAXCOUNT = 100

#数据在这里输入，依次键入每个城市的坐标
cities = np.array([
        [256, 121], 
        [264, 715], 
        [225, 605],
        [168, 538],
        [210, 455],
        [120, 400],
        [96, 304],
        [10,451],
        [162, 660],
        [110, 561],
        [105, 473]
        ])

def calDist(xindex, yindex):
    return (np.sum(np.power(cities[xindex] - cities[yindex], 2))) ** 0.5

def calPathDist(indexList):
    sum = 0.0
    for i in range(1, len(indexList)):
        sum += calDist(indexList[i], indexList[i - 1])
    return sum    

#path1长度比path2短则返回true
def pathCompare(path1, path2):
    if calPathDist(path1) <= calPathDist(path2):
        return True
    return False
    
def generateRandomPath(bestPath):
    a = np.random.randint(len(bestPath))
    while True:
        b = np.random.randint(len(bestPath))
        if np.abs(a - b) > 1:
            break
    if a > b:
        return b, a, bestPath[b:a+1]
    else:
        return a, b, bestPath[a:b+1]
    
def reversePath(path):
    rePath = path.copy()
    rePath[1:-1] = rePath[-2:0:-1]
    return rePath
    
def updateBestPath(bestPath):
    count = 0
    while count < MAXCOUNT:
        print(calPathDist(bestPath))
        print(bestPath.tolist())
        start, end, path = generateRandomPath(bestPath)
        rePath = reversePath(path)
        if pathCompare(path, rePath):
            count += 1
            continue
        else:
            count = 0
            bestPath[start:end+1] = rePath
    return bestPath
    
    
def draw(bestPath):
    ax = plt.subplot(111, aspect='equal')
    ax.plot(cities[:, 0], cities[:, 1], 'x', color='blue')
    for i,city in enumerate(cities):
        ax.text(city[0], city[1], str(i))
    ax.plot(cities[bestPath, 0], cities[bestPath, 1], color='red')
    plt.show()
    
def opt2():
    #随便选择一条可行路径
    bestPath = np.arange(0, len(cities))
    bestPath = np.append(bestPath, 0)
    bestPath = updateBestPath(bestPath)
    draw(bestPath)
    
opt2()

```
运行结果（只取了最后两行输出）：
```python
1511.49908777
[0, 4, 3, 2, 1, 8, 9, 10, 7, 5, 6, 0]
```
![运行结果](http://ou427u8j5.bkt.clouddn.com/2-opt%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.png)

注：因为该算法具有随机性，所以每次运行的结果可能都有所不同，但是大多都是在1500~1600之内，要多运行几次比较一下取较好的结果，我上面给的运行结果就是我在几次运行中挑选得比较好的一次

**关于调参：**
本算法只有COUNTMAX一个参数，用于设置算法的停止条件，当算法已经连续COUNTMAX次没能得到更好的路径时便会停止，经过试验，一般设置大一些效果会比较好，但是大到一定程度的情况下再增大数值算法的效果也不会变得更好。



 