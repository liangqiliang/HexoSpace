---
title: 使用git+hexo+nodejs搭建个人博客
date: 2016-12-30 10:48:53
categories: [Hexo]
tags: [Git,Hexo,NodeJs,nmp]
---

> 这是自己基于Hexo框架搭建的一个个人博客，在这里总结下大致的搭建步骤

<!-- more -->

## 初衷
- 为了有一个更好的载体，而不是简单的文档存储
- 记录开发过程中出现的问题与解决方案，以及学习过程
- 方便交流，方便分享绵薄的知识
- 见证自己的成长

## Git

### Git概述

- Git是一款免费、开源的分布式版本控制系统，用于敏捷高效地处理任何或小或大的项目
- 在此，用git bash进行相应的插件安装、设置、配置及部署，以及为以后发布文章、修改项目带来便利

### Git安装
- 在官网<https://git-scm.com>下载git的安装包，双击后选择安装目录进行安装
- 在目录Bin下找到bash.exe并运行，这是一款基于msys GNU环境的Windows下的命令行工具，有git分布式版本控制工具，也主要用于git。
- 在命令行里输入: `git --version`,出现版本信息等字样，表示安装成功，如下图
	- ![git安装成功信息](/images/git安装成功.png)

## NodeJs

### NodeJs安装
- 在官网<http://nodejs.cn/>下载nodejs安装包，双击后选择安装目录进行安装
- 在git bash下输入: `npm -v`,出现版本信息等字样，表示安装成功，如下图
	- ![nodejs安装成功信息](/images/nodejs安装成功.png)

### nodejs源设置
- npm全称Node Package Manager，是Node.js的模块依赖管理工具。
- npm是随同NodeJS一起安装的包管理工具，能解决NodeJS代码部署上的很多问题，常见的使用场景有以下几种：
	- 允许用户从npm服务器下载别人编写的第三方包到本地使用。
	- 允许用户从npm服务器下载并安装别人编写的命令行程序到本地使用。
	- 允许用户将自己编写的包或命令行程序上传到NPM服务器供别人使用。
- 由于npm的源设在国外，所以可能因为网络原因无法下载软件，所以使用淘宝的提供的源<http://npm.taobao.org/>
- 在git bash下输下以下内容
``` bash
	alias cnpm="npm --registry=https://registry.npm.taobao.org \
	--cache=$HOME/.npm/.cache/cnpm \
	--disturl=https://npm.taobao.org/dist \
	--userconfig=$HOME/.cnpmrc"
```
- 使用`cnpm info express`验证是否可以使用淘宝的cnmp
	- ![cnmp版本信息](/images/cnmp信息.png)
- **注意**:每次重新打开bash使用cnmp命令时都要重新执行上面的命令，因为这是一个临时性的命令

## Hexo安装及创建

### 安装Hexo框架
- Hexo是一款简单、快速、强大的Node.js静态博客框架
- 在git bash下输入 `cnpm install -g hexo-cli`
	- 下载过程因为网络原因速度会有所差池，如果安装失败，重新输入命令，或重新执行上述过程

### 创建Hexo项目
- 在磁盘中创建一个folder
- git bash下输入
	- `cd F:\HexoSpace`(此处为你的目录)
	- `hexo init`
	- `cnmp install`,此处可忽略warn
- 安装完成后，在目录下会有以下项目结构
	- ![hexo目录结构](/images/hexo安装目录.png)
- 输入hexo server即可开启服务，在浏览器中输入localhost:4000，可看见生成的博客页面
	- ![hexo博客页面](/images/hexo博客页面.png)
- 想要停止服务: `Ctrl+C`

### 更换主题
