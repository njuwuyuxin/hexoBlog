---
title: 用Hexo和Github pages快速搭建个人博客
date: 2019-04-09 20:07:32
categories: 环境搭建
tags: 
- hexo
- node
- 指南
---

----

## 为什么要写博客
一直以来都有想写博客的想法，但一方面又觉得自己没有什么技术积累，言之无物，另一方面又担心没有毅力能够坚持下去。终于还是决定先行动起来，即便是记录下日常学习的心得，踩过的坑，也或许对自己对他人有些微帮助

于是今天动手用hexo简单搭建了这样一个静态博客，搭建的过程也并不复杂，感兴趣的朋友可以参照下面步骤搭建一个自己的静态博客

## 开始搭建
### 准备工作
- 首先hexo是基于Node.js实现的，因此我们想要用hexo搭建个人主页，首先要安装Node.js
   - 对于windows用户，建议去官网下载安装包，安装时选择 add to path， 添加环境变量
   - 对于mac用户 可以选择使用nvm进行安装，优点在于可以方便的控制node版本（对于搭建个人博客意义不大）
   `$ curl https://raw.github.com/creationix/nvm/v0.33.11/install.sh | sh`
    安装好nvm后执行
    `nvm install stable`
    安装最新稳定版node

- 安装好node后，为了将其发布在Github pages上，我们还需要安装git
   - 对于windows用户，去官网下载 [git](https://git-scm.com/download/win)，为了方便使用命令行，建议安装git bash
   - 对于mac用户，可以用homebrew进行安装 
   ` brew install git `

### 安装hexo
准备工作完成后，我们就可以安装Hexo了
` npm install -g hexo-cli `
-g 参数指定以全局方式安装

安装好hexo后，便可以在命令行使用hexo指令
```
hexo init <folder>
cd <folder>
npm install
```
其中folder为你想创建的文件夹路径，如果不指定folder，则默认会在当前文件夹创建（要求当前文件夹为空）

新建完成后，文件夹目录结构如下
```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```

- _config.yml为全局配置文件，可以配置网站的基础信息
- scaffolds文件夹存放页面的模版信息
- source文件夹中的_posts文件夹用来存放我们的博文
- themes文件夹存放页面所使用的主题

### 配置网站
到了这里，我们的网站已经初步成型了，为了看到我们网站的具体样子，我们可以执行
`hexo server`
在本地运行一个服务，默认4000端口，信息如下
```
INFO  Start processing
INFO  Hexo is running at http://localhost:4000 . Press Ctrl+C to stop.
```
看到这样的提示，代表已经成功运行了，打开浏览器输入 localhost:4000 即可看到我们的页面

但是此时的网站没有名称，作者等一系列信息，需要我们手动配置

打开根目录下的_config.yml 如下
```
# Hexo Configuration
## Docs: https://hexo.io/docs/configuration.html
## Source: https://github.com/hexojs/hexo/

# Site
title: 
subtitle:
description:
keywords:
author: 
language: zh-Hans
timezone:
```
  可以修改各个字段的值，如标题、作者、语言等等。可以给博客起一个喜欢的名字，并落上自己的署名
  
### 发布文章
  博客配置好后，我们便可以开始书写文章了，用hexo创建一篇新文章也很简单
  `hexo new [layout] <title>`
layout不指定的话默认试用post的布局，默认布局可以在_config.yml中修改
创建好文章后，我们就可以在source/_posts文件夹下找到并编写了，书写博文使用markdown

    文章写好后，我们需要把markdown文件转换成静态的html文件以便显示在网页上，hexo为我们提供了一个简单的指令
`hexo generate`
可以简写为`hexo g` 

在生成好文章后，刷新我们本地打开的博客网站(localhost:4000)，可以看到我们的文章已经可以显示出来啦

### 部署网站
至此我们的博客基本功能已经实现了，但是所有的操作都只能通过本地运行的服务进行查看。为了把博客放到互联网上供所有人浏览，我们还需要将我们的博客部署到服务器上。

一个令人兴奋的消息是，github为我们提供了这样一个静态网站托管的服务，并且完全免费！
我们所需要做的，仅仅是拥有一个github账号，并且创建一个用于维护github page的仓库

- 首先在github上创建一个仓库，仓库名称为 yourName.github.io ，yourName需要替换成你的github昵称
- 如果想要通过ssh验证，需要先在本机生成ssh密钥，将公钥添加到github账户上
- 之后需要配置本地博客网站的部署配置，依然是在_config.yml中，在文件最下方找到deploy字段如下
```
deploy:
  type: 
```
在type字段中填写 git
之后在下一行新增一个字段 repo，填入你刚刚创建的git仓库地址，应该是如下形式
```
deploy:
  type: git
  repo: https://github.com/xxxx/xxx.github.io.git
```
repo字段根据选择的不同协议，可以选择https或者ssh认证

一切都配置完毕后，我们就可以将网站部署到github page上去了！
`hexo deploy`
可以简写为 `hexo d`
首次部署需要进行身份验证，如果采用https协议，需要输入github账号密码。如果采用ssh协议则不需要。

如果没有提示什么错误，稍等片刻，我们在浏览器输入与刚刚创建好的仓库的同名域名 xxx.github.io 即可以看到我们创建好的个人网站了！

### 个性化域名
如果想要为自己的网站设置一个个性化的域名，那么我们需要向域名供应商购买一个域名并且配置相应的dns服务，更多内容可以自行查阅，本文不再过多阐述。

### 相关阅读
[hexo官方文档](https://hexo.io/zh-cn/docs/front-matter)
[github pages官方指南](https://pages.github.com/)
[markdown语法简介](https://www.jianshu.com/p/191d1e21f7ed)











