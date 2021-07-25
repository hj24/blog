---
title: "如何发布自己的Python库"
date: 2019-08-15T15:35:36+08:00
draft: false
tags: ["Python"]
---
<meta name="referrer" content="no-referrer" />

## 前言与简介

我想任何一个有追求的Python开发者在度过基础阶段后都想过发布自己的库，也就是我们常说的造轮子，这是成为一个成熟Python开发者的第一步，在造轮子的过程中，无论是自己的编程能力还是为以后给开源项目贡献代码的能力都会得到很大提升。今天这篇博客，就来带大家从0开始，向PyPI贡献自己的开源库。

<!--more-->

进入正文之前先说一下自己造轮子的步骤：

1. 主体程序设计与编写
2. 编写setup.py
3. 编写使用文档
4. 发布到PyPI

那么，[PyPI](https://pypi.org/)是什么呢？

PyPI (Python Package Index) 是python官方的第三方库的仓库，所有人都可以下载第三方库或上传自己开发的库到PyPI。PyPI推荐使用pip包管理器来下载第三方库。截至目前，PyPI已经有179,387个项目，很多知名项目如Django都发布在上面：

![PyPI主页](https://ws4.sinaimg.cn/large/006tNc79ly1g2xaxmeqslj312n0efdhn.jpg)



## 正文

### 主体程序编写：

一般来说，自己造轮子无非是处于两种目的，一是现有的轮子不能满足自己的需求，二是为了锻炼自己的能力，在这里，我们就不去编写一个有实际用途的轮子了，请根据你自己的需求自行编写，如果你无从下手，这里可以给一个我之前造的小轮子，跨平台统计代码行数的工具作为参考，项目地址：https://github.com/hj24/count-line

项目大体结构：

![项目](https://ws4.sinaimg.cn/large/006tNc79ly1g2xccdox13j30ra04u3z5.jpg)

下面的主体程序代码仅供示例，演示如何上传自己的项目：

1. 新建一个项目文件夹，mypackage，初始化成git仓库。

2. 新建主程序，test.py:

``` python
__version__ = '0.1.0'

"""
实现你自己的轮子的功能
"""

def main():
  pass

if __name__ == '__main__':
    mian()
```

3. 编写完成之后，将其上传至Github，或其他代码托管平台。

### 选择合适的开源证书

任何一个开源项目都应当选择一个开源许可证，没有 License 的内容是默认会被版权保护。所以如果你想要的是让大家都放心使用，就需要选择一个合适的 License ，只有这样才能赋予任何人使用，分享和修改这个软件的权力。

具体如何选择，请看知乎的这篇回答：[传送门](https://zhuanlan.zhihu.com/p/51331026)

总结一下就是：

- MIT 最自由，没有任何限制，任何人都可以售卖你的开源软件，甚至可以用你的名字促销。
- BSD 和 Apache 协议也很自由，跟 MIT 的区别分别是不允许用作者本人名义促销和保护作者版权。
- GPL 最霸道，对代码的修改部分也必须是 GPL 的，同时基于 GPL 代码而开发的代码也必须按照 GPL 发布，
- MPL 相对温和一些，如果后续开发的代码中添加了新文件，同时新文件中也没有用到原来的代码，那么新文件可以不必继续沿用 MPL 。

一般来说，如果选择MIT 协议就可以了。

### 编写setup.py

`setup.py`是每个能从[PyPi](https://pypi.org/)上能下载到的库都有的文件，它是发布的关键所在。

网上的大部分教程都很复杂，新手很难看懂怎么编写，好在[kennethreitz](https://github.com/kenneth-reitz)大神帮我们解决了这个难题，他编写了一个for human的setup.py模板，项目地址：[传送门](https://github.com/kennethreitz/setup.py/blob/master/setup.py)，我们只需要把它复制过来，修改自己项目需要的地方即可，不需要额外的编写setup.cfg等其他文件。

代码请点击传送门查看(131行，就不复制了...)，我们需要重点关注的是如下几个部分：

1. 项目的配置信息：

```python
# Package meta-data.
NAME = 'mypackage'
DESCRIPTION = '填写你的项目简短描述.'
URL = 'https://github.com/你的github账户/mypackage'
EMAIL = 'me@example.com'	# 你的邮箱
AUTHOR = 'Awesome Soul'		# 你的名字
REQUIRES_PYTHON = '>=3.6.0'	# 项目支持的python版本
VERSION = '0.1.0'			# 项目版本号
```

2. 项目的依赖库(没有就不填)：

```python
# What packages are required for this module to be executed?
REQUIRED = [
    # 'requests', 'maya', 'records',
]
```

3. setup:

   - 这里大部分内容都不用你填，只有以下几个注意点

   - 需要注意的是`long_description`这里默认是你项目的README.md文件

   - 注释掉的`entry_points`部分是用来生成命令行工具或者GUI工具的（理论上是跨平台的），比如这里我生成了一个test的命令来代替test.py的main函数，安装成功以后就可以直接使用“test”命令：

     ```python
     entry_points={
         'console_scripts': ['test=test:main'],
     },
     ```

   - 如果你的项目文件夹下只有一个py文件来实现你的功能的话，需要将`packages=find_packages(exclude=["tests", "*.tests", "*.tests.*", "tests.*"]),`注释掉，然后取消py_modules的注释并进行相应修改。

```python
setup(
    name=NAME,
    version=about['__version__'],
    description=DESCRIPTION,
    long_description=long_description,
    long_description_content_type='text/markdown',
    author=AUTHOR,
    author_email=EMAIL,
    python_requires=REQUIRES_PYTHON,
    url=URL,
    packages=find_packages(exclude=["tests", "*.tests", "*.tests.*", "tests.*"]),
    # If your package is a single module, use this instead of 'packages':
    # py_modules=['mypackage'],

    # entry_points={
    #     'console_scripts': ['mycli=mymodule:cli'],
    # },
    install_requires=REQUIRED,
    extras_require=EXTRAS,
    include_package_data=True,
    license='MIT',
    classifiers=[
        # Trove classifiers
        # Full list: https://pypi.python.org/pypi?%3Aaction=list_classifiers
        'License :: OSI Approved :: MIT License',
        'Programming Language :: Python',
        'Programming Language :: Python :: 3',
        'Programming Language :: Python :: 3.6',
        'Programming Language :: Python :: Implementation :: CPython',
        'Programming Language :: Python :: Implementation :: PyPy'
    ],
    # $ setup.py publish support.
    cmdclass={
        'upload': UploadCommand,
    },
)
```

至此setup.py就完成了。

### 编写使用文档

一个好的项目，是需要有一个条理清晰的文档的，至于如何编写，就看你在README.md里怎么发挥了。

### 发布

1. 先去 https://pypi.org 注册一个属于自己的账号，记下账号密码。
2. 由于我们之前编写好了setup.py，这里只要在项目的文件夹下运行`python setup.py upload`即可，中间需要你输入账号密码。

至此，一个项目已经上传完毕了，只需`pip install mypackage`即可使用，下面扩展一下聊聊，怎么进行后续的维护。

### 项目的维护升级

1. 有更新升级之后，首先要运行如下命令删除dist文件夹中的旧版本打包文件，然后生成新文件：

   ```
   sudo python setup.py sdist
   ```

2. 之后，输入以下命令，上传新版本即可：

   ```
   python setup.py upload
   ```

## 结语

写到这里，一个项目的完整发布与维护流程已经结束了，希望能帮助到同时Python开发者的你。

插个题外话，如果大家对开头的自动统计代码行数的工具满意的话，欢迎给个star...