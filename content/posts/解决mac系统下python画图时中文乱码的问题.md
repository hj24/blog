---
title: "解决mac系统下python画图时中文乱码的问题"
date: 2019-06-05T15:34:00+08:00
draft: false
tags: ["踩坑记录"]
---
在mac上用python的matplotlib库画图的人大概都遇到过中文显示乱码的问题，网上大多博客都是采用下载新的字体库解决的，不过这里提供一种更简单，不费事的方法。
<!--more-->


## 解决
百度或谷歌matplotlib库绘图时产生中文乱码问题，得到的最多的答案就是下面几行代码：

```python
import numpy as np
import matplotlib.pyplot as plt
plt.rcParams['font.sans-serif'] = ['SimHei']
```

很明显，这是因为mac下没有SimHei字体库，于是大多数教程都叫你怎么下载SimHei字体怎么放到mac的字体库，以及配置matplotlib的字体库，可是这些教程大都是几年前的，有的已经失效，有的则过于复杂。
于是...为什么要费这些功夫呢，直接找找mac底下有哪些支持中文的字体库不就好了嘛...然后我还真找到了`Arial Unicode MS`，亲测可用，代码如下：

```python
plt.rcParams['font.sans-serif'] = ['Arial Unicode MS']
```

不信你可以用下面的完整的绘图代码试试看：

## 测试代码

```python
import numpy as np
import matplotlib.pyplot as plt
plt.rcParams['font.sans-serif'] = ['Arial Unicode MS']

plt.figure()
names = ['5', '10', '15', '20', '25']
x = range(len(names))
y = [0.855, 0.84, 0.835, 0.815, 0.81]
y1=[0.86,0.85,0.853,0.849,0.83]
plt.plot(x, y, marker='o', mec='r', mfc='w',label=u'y=x^2曲线图')
plt.plot(x, y1, marker='*', ms=10,label=u'y=x^3曲线图')
plt.legend()  # 让图例生效
plt.xticks(x, names, rotation=45)
plt.margins(0)
plt.subplots_adjust(bottom=0.15)
plt.xlabel(u"time(s)邻居") #X轴标签
plt.ylabel("RMSE") #Y轴标签
plt.title("我就看看中文能不能用") #标题
plt.show()
```

