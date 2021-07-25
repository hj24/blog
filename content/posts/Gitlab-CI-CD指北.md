---
title: "Gitlab CI CD指北"
date: 2020-02-05T19:50:30+08:00
draft: false
tags: ["Devops"]
---

<meta name="referrer" content="no-referrer" />
之前在公司倒腾过基于Gitlab的CI/CD，现在重新记录一遍流程，做个备忘。

一个典型的流程是这样的：

> 提交代码触发CI pipeline -> 安装依赖、编译、自动化测试 -> fix bug 
> -> 回到上一步 -> Review代码 -> 合并到发布分支 -> 触发CD -> 部署代码、发布

基本上更复杂一些的流程都是在这个基础上增改了，下面来走一遍这个流程。

<!--more-->
- 如果你有账号有项目，想直接知道怎么开始CI/CD，请跳过下面这一节

## 准备
一般公司用的Gitlab都是自己搭建在内网的，不过好在Gilab提供了一个30天的免费体验活动，可以体验到Gitlab的所有功能，这篇教程就用一个新注册的账号，从头到尾走一遍CI/CD的流程。

1. 先去注册一个账号:

![注册界面](https://tva1.sinaimg.cn/large/006tNbRwgy1gblhoz4zzxj315p0rhtel.jpg)

2. 创建一个test项目:

![创建项目](https://tva1.sinaimg.cn/large/006tNbRwgy1gblhq7x8fzj315t0rhjxy.jpg)

3. 把自己的公钥添加到gitlab的SSH keys里，这就不用我多说了，不知道的可以参考这个教程:

> https://www.liaoxuefeng.com/wiki/896043488029600/896954117292416

4. 关于概念: [参考这篇](https://www.jianshu.com/p/306cf4c6789a)，本文会穿插着提及一些概念，但重点仍放在实现这一套流程上。上面这个博客写的比较清晰，当时搞CI/CD时参考过不少。

5. 项目内容准备:
- 拿一个简单的Python项目做演示
- 里面包含了依赖安装，测试，部署这三个过程，基本涵盖了CI/CD需要的基本流程

```
目录结构:
  test/
      app.py
      test_app.py
      requirements.txt
      README.md
```

**app.py**里面是[leetcode 70](https://leetcode-cn.com/problems/climbing-stairs/)的解决代码
加了个run函数来模拟部署时执行开启程序的过程，类比于django的`run server`命令:

```python
class Solution:
        
    def climbStairs(self, n):
        if n == 1:
            return 1
        if n == 2:
            return 2
        a = 1; b = 2; c = 0
        for i in range(3, n + 1):
            c = a + b
            a, b = b, c
        return c

def run():
    Solution().climbStairs(10)
```

**test_app.py**里是对app的测试，测试用的是pytest，里面包含了3个简单的测试样例:
```python
from app import Solution

def test_app():
	sol = Solution()

	assert sol.climbStairs(2) == 2
	assert sol.climbStairs(3) == 3
	assert sol.climbStairs(10) == 89
```

pytest测试运行这个文件很简单，只要在当前目录下运行`pytest test_app.py`就可以了
准备工作就这么多，下面开始正式的流程

## 开始配置CI/CD

开始之前需要明确一下，一个完整的CI/CD流程如下:
> 编写好`.gitlab-ci.yml` -> 本地push代码触发runner -> build-test-deploy
> 正常来说deploy时是要把测试完毕的代码pull到服务器的，而我们这个演示runner是安装在本地的
> 所以其实过程是本地开发，部署到本地的另一个端口，不过对于演示来讲，没啥区别

1. 安装runner
runner是CI的运行环境。一般每个项目都会有一个`.gitlab-ci.yml`脚本来定义CI时要执行的一系列操作，而runner扮演的角色就是在push时被触发，然后在本地执行`.gitlab-ci.yml`中定义的操作，runner肯定是要安装到本地的，在本地与gitlab服务器通信。

> mac: `brew install gitlab-runner`

安装后`sudo chmod +x /path/to/gitlab-runner`给它添加执行权限
一般mac安装的地址是: `/usr/local/bin/gitlab-runner`

2. 注册runner

> gitlab的项目->settings->CI/CD 

下拉到如下页面，可以看到Specific Runners底下有两个标红的信息url和token，这是等会注册时要用的
![](https://tva1.sinaimg.cn/large/006tNbRwgy1gbllkvpkruj31340lcaex.jpg)

运行: `gitlab-runner register`开始注册，会要求你填写如下信息，相关的解释我写在注释中了

```bash
# 填写gitlab-ci的url，就是前面在页面上获得的url
Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
https://gitlab.com/

# 填写token，写你获取的就好了
Please enter the gitlab-ci token for this runner:
xxx

# 给runner的描述，看情况写
Please enter the gitlab-ci description for this runner:
[macbookdeMacBook-Air-2.local]: test

# tag，后续的.gitlab-ci.yml要区分用哪个runner执行时需要它
Please enter the gitlab-ci tags for this runner (comma separated):
v1 
Registering runner... succeeded                     runner=By1ur7Pe

# 填写executer，由于我们是直接在mac下安装的，没有用docker什么的，直接填shell就好
# 当然这个填什么要看你的环境
Please enter the executor: kubernetes, docker-ssh, parallels, virtualbox, ssh, docker+machine, docker-ssh+machine, custom, docker, shell:
shell
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded! 
```

注册完毕之后，可以`cat ~/.gitlab-runner/config.toml`查看之前注册的配置

3. 运行
执行如下命令运行runner:
```
cd ~
gitlab-runner install
gitlab-runner start
```
运行后回到gitlab的settings页面，看到刚才注册的runner状态变绿了，那么说明执行成功：
![](https://tva1.sinaimg.cn/large/006tNbRwgy1gblm0psuv0j30cm054aae.jpg)

## 开始CI/CD
开始之前回顾一下GitLab CI/CD的流程:
1）当我们本地开发完push代码到服务器时触发pipeline来进行构建任务
2）在pipeline中gitlab runner来运行`.gitlab-ci.yml`内编写的脚本
3）`.gitlab-ci.yml`中的任务是按stages分配的，一个stage结束了才能进入下一个stage，若其中一个失败则后续的都不会执行，整个CI流程也会失败，有一个注意点：stages不能定义具体的执行逻辑，如何执行任务是否job定义的，然后在job中绑定上对应的stage以此来执行任务
4）job的名字随意定义，主要绑定对了相应的stage就能正常执行

下面来看具体例子，相关解释写在注释中:

1. 编写`.gitlab-ci.yml`
**ps: 要注意该文件需要符合yaml语法**

```yaml
# 定义CI/CD流程需要的stages，以下的build-test-deploy顺序执行
stages:
  - build
  - test
  - deploy

# 定义各个流程需要的job
## 构建项目的job，指定运行的runner的tag为v1，就是我们刚才配置的
## 在这个build任务里，我们简单的安装了项目所需要的依赖
## 由于runner的executor是shell，所以会在runner所在的机器上创建一个临时的build文件夹存放代码
job_build:
  stage: build
  tags:
    - v1
  script:
    - echo "installing"
    - pip install -r requirements.txt -i https://mirrors.aliyun.com/pypi/simple/
    - echo "finish installing"

## 测试阶段的job，使用pytest运行单元测试
job_test:
  stage: test
  tags:
    - v1
  script:
    - echo "testing"
    - pytest test_app.py
    - echo "finish testing"

## 部署阶段，在脚本开始之前先清空之前部署好的代码文件，注意，这并不是必须的，仅仅只是针对这个示例项目而言
## 真实项目大都是用git pull更新代码，docker构建镜像之类的
## 清空旧代码之后git clone拉取最新代码到新目录test_deploy下，进入到项目根目录下运行app.py
job_deploy:
  stage: deploy
  tags:
    - v1
  only: 
    - master
  before_script:
    - cd ~/desktop
    - rm -rf test_deploy/
    - mkdir test_deploy
    - cd test_deploy
  script:
    - echo "deploying"
    - git clone git@gitlab.com:mambahj/test.git
    - cd test
    - python app.py
    - echo "finish deploying"
```

2. push代码，执行CI/CD流程

> git add -> git commit -> git push

回到Gitlab，可以看到pipeline被触发，runner开始执行CI/CD任务
![](https://tva1.sinaimg.cn/large/006tNbRwgy1gblux8a67sj30w805jdgr.jpg)
最后，流程执行完毕，去test_deploy目录可以看到，代码被正常拉取和执行
![](https://tva1.sinaimg.cn/large/006tNbRwgy1gbluyqgm01j30r103qt9a.jpg)

## 一些思考
后来去百度面试的时候问到对CI和CD的理解，CI阶段应该做些什么，CD阶段应该做什么，感觉这个问题用来结束这篇博客不错。

CI也就是持续集成，我的理解是这个阶段在代码被合并之前，应该负责做好完善的测试和构建检查工作，尽可能的发现并解决问题来保证合并后的代码的质量。
CD指的是持续交付，也就是我们的应用发布出去的过程，这个阶段主要要做好自动构建和发布代码的工作，我的理解是这一阶段的主要目的是为了快速的进行产品的交付，缩短发布周期吧，如果没有CD过程，以前发布时都是人工跑命令，跑脚本，出错了去定位，公司之前有个其它组的老哥，每次发代码之前都会留下来加班...
当然，缩短了发布周期之后即会带来好处，也会带来坏处，好处是开发效率的提高，开发人员不用把精力放在繁琐重复的发代码流程上，而且产品的反馈也更快，相应的完善速度也会更快，产品肯定会做的往好的方向发展的。但坏处就是，开发团队的压力肯定就会大了，平时半月或者一个月一发，现在每周每天都会发...等着加班吧...

## 参考
1. [花椒前端基于 GitLab CI/CD 的自动化构建、发布实践](https://zhuanlan.zhihu.com/p/69513606)
2. [简书-Gitlab CI/CD的执行流程](https://www.jianshu.com/p/306cf4c6789a)