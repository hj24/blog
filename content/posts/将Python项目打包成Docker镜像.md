---
title: "将Python项目打包成Docker镜像"
date: 2020-01-31T15:36:42+08:00
draft: false
tags: ["Devops"]
---

<meta name="referrer" content="no-referrer" />
之前写了一篇 Dockerfile编写指南，不过没有涉及到部署项目这种相对复杂的操作，最近写毕设需要把项目打包成docker镜像，部署在服务器上，因此写下这篇博客，对上面那篇做一个补充引申。

一个将Python项目打包成docker镜像的基本流程如下:
> 编写好Dockerfile上传至Github/Gitlab -> git拉取代码到服务器 -> build镜像
> -> docker run配置端口映射 -> 设置服务器安全组

<!--more-->

阅读本文，大概需要你掌握git和docker的基本用法。
> ps: 本文默认你的机器已经装好并配置好git，如何安装？👇
> [菜鸟教程](https://www.runoob.com/git/git-install-setup.html)


## 编写项目所需的Dockerfile
- 将项目打包成镜像的Dockerfile一般放在项目根目录下，这样方便后面设置镜像的工作目录等。

1. 基础镜像以及维护者信息
  一个python项目的版本是很重要的，因为从2到3 (这个就不用多说了)，3.4向后的版本 (3.5的协程引入了`async/await`关键字)会有一些不兼容的地方，所以设置基础镜像之前请先查看项目用的python版本，以防不必要的麻烦。
  这里我用的是Python 3.6,3，所以：
  ```dockerfile
  # 以Python3.6.3作为基础镜像
  FROM python:3.6.3

  # 维护者
  MAINTAINER mambahj24@gmail.com
  ```
2. 把项目代码添加到镜像中
  部署项目和之前的Dockerfile有一个不同点就是需要把自己的代码放到镜像中，这里就要引入两个之前没讲的关键字ADD，WORKDIR：
  - 使用ADD，可以把本地文件添加到容器中
  - WORKDIR，类似于cd命令，就是将这个目录设置为镜像的工作目录

  假设你的项目根目录是`/proj`，我们一般需要设置这两步:
  ```dockerfile
  # 将项目文件添加到镜像内的spider文件夹
  ADD . /proj

  # 设置spider目录为工作目录,相当于cd
  WORKDIR /proj
  ```
  **.代表当前目录，也就是Dockerfile所在的目录。**

3. 设置环境变量、端口
  一般项目里肯定是要有环境变量还要设置端口号的，这两个就不多说了：
  ```dockerfile
  ENV var val

  EXPOSE 端口号
  ```
4. 安装环境依赖和项目依赖
  举个例子，我要在镜像中安装好vim和lsof，并且安装好项目的依赖requirements.txt文件，最好还要有阿里云的镜像加速，这应该是一个python项目部署很常见的场景了:

  ```dockerfile
  # build镜像时运行的命令，安装vim，以及项目依赖
  RUN apt-get update && apt-get install vim -y \
      && apt-get install lsof -y && apt-get clean \
      && pip install -r requirements.txt -i https://mirrors.aliyun.com/pypi/simple/
  ```

5. 最后也是最重要的，`docker run`构建容器时默认执行的命令
  前面已经说过，要用CMD，我们需要在开启容器时替我们运行好启动项目的命令。
  不过CMD有一个局限就是，一个Dockerfile中只有最后一个CMD才会生效，所以，如果你的项目启动时需要启动很多个命令，我的建议是写一个shell脚本，把命令放进去，然后在CMD中用`["sh", "xxx.sh"]`来执行。

  这里就不得不提容器中应用在前台执行和后台执行的问题。这是初学者常出现的一个混淆。

  以下引用一段docker的文档：

  > Docker 不是虚拟机，容器中的应用都应该以前台执行，而不是像虚拟机、物理机里面那样，用 systemd 去启动后台服务，容器内没有后台服务的概念。
  > 一些初学者将 CMD 写为：
  > `CMD service nginx start`
  > 然后发现容器执行后就立即退出了。甚至在容器内去使用 systemctl 命令结果却发现根本执行不了。这就是因为没有搞明白前台、后台的概念，没有区分容器和虚拟机的差异，依旧在以传统虚拟机的角度去理解容器。
  > 对于容器而言，其启动程序就是容器应用进程，容器就是为了主进程而存在的，主进程退出，容器就失去了存在的意义，从而退出，其它辅助进程不是它需要关心的东西。

  我来解释一下，使用service就是要以守护进程在后台执行程序，类似的还有nohup命令，但是`CMD service nginx start`其实在docker中是被这样执行的：

  ```dockerfile
  CMD ["sh", "-c", "service nginx start"]
  ```
  主进程其实是sh，当`service nginx start`结束了之后，sh作为主进程退出，容器也会跟着退出。所以使用CMD命令最主要的一点就是要让容器中的应用在前台执行，总之，不能让主进程退出。

  所以，执行多个命令时，这些命令最好都要直接在前台运行，但如果几个命令都是阻塞式的：
  比如一个是`django runserver`一个是celery启用worker，在不用docker时好办，直接nohup或者其他方式扔到后台执行，互不干扰。
  但在构建容器时，就要考虑上述的因素，既不能全部后台执行，也不能全部前台执行（因为是阻塞式的命令，只有一个执行完毕才轮到下一个），我认为最好的方案就是前面的命令全后台执行，最后一个命令让它阻塞当前线程，不让主进程退出，这样就既能保证所有目录都在执行，并且docker容器不会退出。

  举个例子，一个启动用的shell脚本，xxx.sh：
  ```bash
  #!bin/bash
  nohup command_a > a.log 2>&1 &
  nohup command_b > b.log 2>&1 &
  command_c
  ```

  Dockerfile中应该这样写:
  ```dockerfile
  CMD ["sh", "xxx.sh"]
  ```
最后，放一个整体的模板:
```dockerfile
# 以Python3.6.3作为基础镜像
FROM python:3.6.3

# 维护者
MAINTAINER mambahj24@gmail.com

# 将项目文件添加到镜像内的文件夹
ADD . /proj

# 设置代码目录为工作目录,相当于cd
WORKDIR /proj

# 设置环境变量
ENV var val

# 将端口暴露出去
EXPOSE port

# build镜像时运行的命令，安装vim，以及项目依赖
RUN apt-get update && apt-get install vim -y \
    && apt-get install lsof -y && apt-get clean \
    && pip install -r requirements.txt -i https://mirrors.aliyun.com/pypi/simple/

# docker run构建容器时会运行的命令，用于启动服务
CMD ["sh", "/proj/xxx.sh"]
```

## 构建，使用镜像
1. 进入Dockerfile所在的目录
2. 构建镜像: `docker build -t images:version .`
3. 构建容器: `docker run --name name -p outside_port:inside_port -d images:version`
4. 查看是否开启成功: `docker ps -a`

更进一步，进入容器中查看端口是否开放:
1. 进入正在运行的容器: `docker exec -it name /bin/bash`
2. 查看端口使用情况: `lsof -i:inside_port`
3. 退出容器: `exit;`注意: exec进入的容器，退出后不会关闭容器，但attach会

## 参考
1. [CMD 容器启动命令](https://yeasy.gitbooks.io/docker_practice/image/dockerfile/cmd.html)
2. [Dockerfile编写指南](https://hj24.github.io/2020/01/30/Dockerfile%E7%BC%96%E5%86%99%E6%8C%87%E5%8D%97/)
3. [简书-Python项目打包成docker镜像](https://www.jianshu.com/p/fe81ca1bccc6)
