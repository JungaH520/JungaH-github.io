---
title: 我的第一个个人博客
tag: 博客
---
最近一直有个想搞个个人博客玩玩的想法，今天在网上看到一个静态博客框架Hexo，于是看教程学着搭建了一个，现在看起来很成功，哈哈！开启我的博客之旅吧！

<div align=center>
    ![gif](http://www.w3cplus.com/sites/default/files/blogs/2017/1702/understanding-flexbox-3.gif)
</div>

<!-- more -->

## 安装前系统所需要的工具

### 1、Git

Git是一个开源的分布式版本控制系统，可以有效、高速的处理从很小到非常大的项目版本管理。Hexo就是将你的博客项目上传托管到git,通过github来访问你的个人博客了。

### 2、Node

因为Hexo是一款基于Node.js的静态博客框架，所以我们还需要安装好它。


### 3、Hexo

安装好Git和Node.js之后，轮到我们今天的主角登场了~先打开Git Bash，然后输入下面这条命令来安装Hexo：

``` bash
$ npm install hexo-cli -g
``` 

安装完输入以下命令查看版本信息：

``` bash
$ hexo -v
``` 

如果出现类似以下内容的则安装成功

``` bash
hexo: 3.2.2
hexo-cli: 1.0.2
os: Darwin 16.1.0 darwin x64
http_parser: 2.7.0
node: 6.9.1
v8: 5.1.281.84
uv: 1.9.1
zlib: 1.2.8
ares: 1.10.1-DEV
icu: 57.1
modules: 48
openssl: 1.0.2j
``` 

接下来就可以用Hexo愉快的zhuangbility了！！！

## 创建属于自己的博客

### 初始化hexo创建博客模板

在本地创建一个文件夹，比如blog,进入文件夹执行以下命令：

``` bash
 $ hexo init
``` 
执行完成之后可以在blog文件下看到博客相关的配置文件~~~

### 配置github

[建立*Repository*](http://blog.csdn.net/yuexianchang/article/details/53431703)，建立与你用户名对应的仓库，仓库名必须为【your_user_name.github.io】，然后与本地blog建立关联，
关联之前首先配置一下blog根目录下的_config.yml文件：

``` bash
 deploy:
     type: git
     repo: 你创建的博客所在的仓库github路径
     branch: master
``` 

然后执行以下命令：其中hexo g是生成静态页面至public目录，hexo deploy #将.deploy目录部署到GitHub

``` bash
$ npm install hexo-deployer-git --save
$ hexo g
$ hexo d
``` 

一些常用的命令：

``` bash
    hexo new "postName" #新建文章

    hexo new page "pageName" #新建页面

    hexo generate #生成静态页面至public目录

    hexo server #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）

    hexo deploy #将.deploy目录部署到GitHub

    hexo help # 查看帮助

    hexo version #查看Hexo的版本
``` 

至此，你可以打开你的个人博客路径username.github.io，就可以看到你的博客了。

>[更加详细的教程前往hexo的中文官网查看](https://hexo.io/zh-cn/)
