---
title: "Hugo使用指北"
date: 2020-11-14T19:54:13+08:00
draft: false
tags: ["踩坑记录"]
---
<meta name="referrer" content="no-referrer" />
从 hexo 迁移到了 hugo，从安装到部署，记录一下使用流程。

<!--more-->

## 安装
mac 用户: `brew install hugo`，其他用户参考[这里](https://www.gohugo.org/)

## 生成站点
使用下面的命令生成你的博客框架:
```shell
hugo new site /path/to/site
```

## 安装皮肤
中文站有个[皮肤列表](https://www.gohugo.org/theme/)，不过比较少，感兴趣可以去看看。

如果你觉得不够，可以找一些其他的第三方主题，知乎有个相关的问题：[有哪些好看的hugo主题](https://www.zhihu.com/question/266175192)，一般主题里都会写怎么使用，以我安装的zozo主题为例：
```shell
# 进入站点目录
cd /path/to/site
# 获取主题到 themes目录下
git clone https://github.com/varkai/hugo-theme-zozo themes/zozo
# 修改 config.toml 文件
```

## 修改模板，定制你的需求

虽然你千挑万选了一个好看的模板，但毕竟每个人的审美是不同的，这个模板难免有些地方不合你心意，这时你就可以尽情调教它了...下面就列举一些常见的修改项:

**PS: 这些模板一般在`/path/to/site/themes/{theme}/layouts/partials/`中，你根据你要改的地方找对应的文件就可以了，比如我要改的是具体文章的显示，在我的主题里那就是在`post.html`中**

### 摘要
hugo 支持指定摘要字数，以我选的zozo主题为例，默认是 100 字作为摘要，这个可以在配置文件中改：
```toml
hasCJKLanguage = true
summaryLength = 120
```
但是它可能经常会断在一些你不想截断的地方，所以也可以直接在正文中使用`<!--more-->`标签，在这个标签上面的是摘要，下面的是正文。但是具体显示还是要看主题的，比如这个zozo主题，默认是在摘要之后跟很多省略号的，导致我改完以后显示成了这样：

![demo](https://tva1.sinaimg.cn/large/0081Kckwgy1gkozpxjnxmj30l80digmx.jpg)

比较丑陋，反正我是忍不了，于是我去模板里找到了下面这段代码：

```html
{{ if .Site.Params.enableSummary }}
    <div class="list">
        <div class="post_content markdown">
            <p>{{ .Summary }}......</p>
        </div>
    </div>
{{ end }}
```
去掉了`<p>{{ .Summary }}......</p>`里面的省略号就可以了。

### 添加 gittalk
旧博客用了 gittalk 做评论系统，这里得把它迁移过来，不过我没有选择直接改主题的框架，hugo 有个很优秀的特点是它的主题和你站点的目录结构是一样的，如果你需要修改主题，也可以直接在你站点对应的目录下添加内容，覆盖主题，这就为我们修改主题提供了很大的主观能动性。

比如我要修改的是评论模块，这个在主题里的路径是：`/path/to/site/themes/{theme}/layouts/partials/comments.html`，我只需要再站点目录的`/path/to/site/layouts/partials/comments.html`写对应的内容就能覆盖这一部分了。

下面就给个例子，不同模板得看着自己的css样式啥的改成对应的，不过gittalk接入的代码都是一样的：
```html
<!-- gitalk -->
<div class="doc_comments">
	<div id="gitalk-container"></div>
</div>
<link rel="stylesheet" href="https://unpkg.com/gitalk/dist/gitalk.css">
<script src="https://unpkg.com/gitalk/dist/gitalk.min.js"></script>

<script type="text/javascript">
let gitalk = new Gitalk({
  clientID: '换成你自己的',
  clientSecret: '换成你自己的',
  repo: 'xxx.github.io',
  owner: 'xxx',
  admin: ['xxx'],
  id: '{{ .Params.Date }}',      // Ensure uniqueness and length less than 50
  distractionFreeMode: false,  // Facebook-like distraction free mode
  title : '{{ .Params.Title }}',
  labels : []
});
gitalk.render('gitalk-container')
</script>
```

接入 gittalk 之前要去申请 Github OAuth App，可以点[这里](https://github.com/settings/applications/new)申请，几个需要填写的内容：

| 字段 | 内容 | 备注 |
| --- | --- | --- |
| Application name | 取一个名字 | 填写应用名称 |
| Homepage URL | https://你的域名 | 主页地址 |
| Application description | 描述，自己写就可以了 | 备注 |
| Authorization callback URL | 和上面的域名一样 | 回调地址 |

注册完之后拿着你的 clientID 和 clientSecret 填到上面，在配置里把原主题自带的评论选项打开，就可以完成覆盖了:

```toml
[params.valine]
  enable = true
  appId = ""
  appKey = ""
  placeholder = " "
  visitor = true
```

### 添加文章版权
TODO，不太懂前端的那一套，等有机会学了改一下样式，把这个添加进去。

## 运行与部署
运行`hugo server --theme=zozo --buildDrafts`在本地访问`http://localhost:1313`查看效果。

### 部署
这里我会把它部署到 github pages，并且绑定上自己的域名。

绑定到 github pages 很简单，按下面这么做，把 theme 和 baseUrl 换成你自己的：
```shell
# 这一步会生成你的站点到 public 目录下 
hugo --theme=zozo --baseUrl="https://hj24.github.io/"

# 进入 public 把它初始化成 git 仓库，并关联你的远程仓库地址
cd /path/to/site/public
git init
git remote add origin https://github.com/hj24/hj24.github.io.git
git add -A
git commit -m "first commit"
git push -u origin master
```

这一步做完你就可以访问 `https://xxx.github.io` 访问你的站点了。如果你有自己的域名，想实现自己的域名访问，那么你还需要额外做两步：

1. 在你的域名服务商哪里配置一条 CNAME 记录，CNAME 简单来说就是别名，让你的域名解析到另一个域名上，我们要做的就是把自己的域名配一条 CNAME 指向 `xxx.github.io`域名。

2. 在 github pages 设置里填上 `custom domain`选项，并勾上 `Enforce https`（可选）

然后就完事了，你就可以愉快的用自己的域名访问自己的 hugo 博客了。

## 参考
1. [y4er的如何修改模板的博客](https://y4er.com/post/hello-world/)
2. [hugo中文文档](https://www.gohugo.org/)
3. [hugo 添加 gittalk 评论系统](https://xbc.me/add-gittalk-to-hugo/)