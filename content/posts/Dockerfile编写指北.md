---
title: "Dockerfile编写指北"
date: 2020-01-30T15:34:40+08:00
draft: false
tags: ["Devops"]
---

<meta name="referrer" content="no-referrer" />
刚开始接触docker时大家基本都是通过`docker pull`命令来拉取镜像，在此基础上`docker exec/run`这些命令，进入容器做一些配置上的修改以此来构建一个容器，而Dockerfile就是一个一劳永益的构建镜像的方法，通过编写Dockerfile来定制自己的镜像。

说白了，就是类似于Python项目的requirements.txt文件，你可以在里面写上自己需要的依赖包，然后安装构建自己项目的依赖:
> requirements.txt => Dockerfile
> pip install => docker build

这篇博客会以构建一个简单的Postgres镜像为例，讲一讲用Dockerfile来定制自己的镜像的过程。

<!--more-->
# 定制一个Postgres镜像
## 一些基本概念
**如果你已经了解过Dockerfile的语法，可以直接跳过这一节**

网上找了张图，可以参考一下:
![Dockerfile图解](https://tva1.sinaimg.cn/large/006tNbRwgy1gbexi2raezj30g608pq9w.jpg)

Dockerfile是有它的基本语法的，它由一行行命令语句组成，其中注释以#开头，并且，一个完整的Dockerfile一般包含下面这几部分：

1. 基础镜像
  由`FROM 基础镜像`这个命令来使用，标志着我们定制的镜像是以哪个镜像作为基础进行制作的。
  举个栗子: `FROM postgres:latest`

2. 维护者信息
  这里要写上Dockerfile编写者的信息，一般写上自己的邮箱或者nickname就可以，用法是`MAINTAINER 个人信息`。
  栗子: `MAINTAINER mambahj24@gmail.com`

3. 镜像的操作命令
  当我们需要定制一个镜像的时候，肯定是要运行一些命令（安装一些依赖，修改一些配置等等）来对基础镜像做一些修改的，一般使用`RUN 具体命令`来操作，RUN的默认权限是sudo。
  需要注意的是，如果你需要执行多个RUN操作，那最好把它们合并在一行 (**用&&连接**)，因为每执行一次RUN就会在docker上新建一层镜像，所以分开来写很多个RUN的结果就是会导致整个镜像无意义的过大膨胀。
  正确的做法:
  ```bash
  RUN apt-get update && apt-get install vim
  ```

  不提倡的做法：
  ```bash
  RUN apt-get update
  RUN apy-get install vim
  ```

4. 容器启动时执行的命令
  需要用CMD来执行一些容器启动时的命令，注意与RUN的区别，CMD是在`docker run`执行的时候使用，而RUN则是在`docker build`的时候使用，还有，**划重点**，一个Dockerfile只有最后一个CMD会起作用。
  栗子：
  ```bash
  CMD ["/usr/bin/wc","--help"]
  ```

除了以上这4个部分，ENV（设置环境变量）、EXPOSE（暴露容器内部端口）也很常用，用法分别是：
```
ENV <KEY> <VALUE>

EXPOSE 端口号
```

## 开始定制一个镜像
开始定制postgres镜像之前，我有一个简单的需求：定制的镜像中要装好vim（docker pull 拉取的默认镜像是没有的）并且要设置好我需要的配置。

接下来就以这个简单的需求来开始定制了。

1. 首先，我们肯定是要在docker提供的postgres镜像上进行定制，当然你要是费心费力的从ubuntu镜像从头安装配置postgres也是可以的，不过实在没必要。。。浪费时间。。

1) `docker search postgres`查找已有的镜像，选一个版本作为基础镜像，这里我直接选了最新的`postgres:latest`。

2) 写好FROM和MAINTAINER
```dockerfile
# Version 1.0
FROM postgres:latest

MAINTAINER mambahj24@gmail.com
```

2. 拉取的镜像里是没有装vim的，因此我们定制时要装好vim，继续写RUN，在`docker build`时运行好安装vim的命令

```dockerfile
# Version 1.0
FROM postgres:latest

# 维护者
MAINTAINER mambahj24@gmail.com

# 镜像操作命令
RUN apt-get update && apt-get install vim -y
```

3. 除了给镜像安装vim之外，我还需要在镜像定制时设置好postgres的密码，并暴露出postgres的默认端口5432：

```dockerfile
# Postgre image for graduation design
# Version 1.0

# 基础镜像 最新的postgres
FROM postgres:latest

# 维护者信息
MAINTAINER mambahj24@gmail.com

# 镜像操作命令
RUN apt-get update && apt-get install vim -y

# 设置环境变量
ENV POSTGRES_PASSWORD 你自己设置的密码

# 暴露端口
EXPOSE 5432
```
至此我们需要定制的这个Dockerfile就完成了，下面就是开始build了。

4. 写好了Dockerfile之后，就要使用`docker build`来定制了，该命令可以指定Dockerfile的路径

关于这个路径，我一般是这么安排的，分为两类：
1）第一类，项目中的Dockerfile
```
proj/
    otherfiles
    ...
    Dockerfile
    ...
    otherfiles
```
这一类直接在项目根目录下运行`docker build .`来根据上下文找到Dockerfile, 注意：可以通过.dockerignore文件排除上下文目录下，不需要的文件和目录。

2）第二类，构建不以项目为单位的，我一般会把目录结构这么划分：
```
dockerfiledir/
    imagesname1/
        Dockerfile
    imagesname2/
        Dockerfile
    ...
    imagesnameX/
        Dockerfile
```
然后通过`docker build -f /path/to/Dockerfile`来构建。

好了，言归正传，我们运行`docker build`来构建我们定制好的镜像：

`docker build -t pgsql:v1 -f /path/to/Dockerfile`

**补充一下，-t参数指镜像的标签信息，:前的部分为REPOSITORY名，:后面是TAG名**

在运行Dockerfile内的命令之前，首先会对它的语法进行检查，如果出错，会返回相关信息，没有问题的话会继续build下去，运行完毕之后可以运行`docker images`查看有没有build成功：
```bash
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
pgsql               v1                  268d1c521b18        16 seconds ago      446MB
menginx             latest              01ecef387cfa        4 weeks ago         177MB
postgres            latest              ec5d6d5f5b34        4 weeks ago         394MB
hello-world         latest              fce289e99eb9        13 months ago       1.84kB
```

成功之后运行:
```bash
docker run --name 容器名 -e POSTGRES_PASSWORD=密码 -p 5431:5432 -d pgsql:v1
```
把里面的容器名和密码改成你自己要设置的，就可以运行postgres容器了，设置完毕之后可以用pgAdmin或navicate或者其他能连接postgresql的工具连接看看。

最后，`docker ps -a`检查一下，看看该容器是否运行起来。

**注意，如果部署在服务器上可能需要设置安全组（类似于防火墙）将端口暴露给外网，来让外网访问那台服务器上某个端口上运行的程序。**

至此，镜像已经定制完毕，希望对大家有所帮助。

## 参考
1. [知乎-如何编写最佳的Dockerfile](https://zhuanlan.zhihu.com/p/26904830)
2. [docs.docker.com-Docker最佳实践-英文](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
3. [dockerfile最佳实践-中文](https://yeasy.gitbooks.io/docker_practice/appendix/best_practices.html)
4. [cnblogs-编写Dockerfile](https://www.cnblogs.com/liuyansheng/p/6098470.html)
5. [阮一峰-Docker入门教程](http://www.ruanyifeng.com/blog/2018/02/docker-tutorial.html)
6. [Dockerfile设置环境变量](https://www.jianshu.com/p/ae634ffb21ff)
7. [docker run 命令参数及使用](https://www.jianshu.com/p/ea4a00c6c21c)
8. [Dockerfile文件详解](https://www.cnblogs.com/panwenbin-logs/p/8007348.html)