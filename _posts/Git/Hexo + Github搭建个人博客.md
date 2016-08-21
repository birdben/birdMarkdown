---
title: "Hexo + Github搭建个人博客"
date: 2016-07-02 16:41:26
tags: [Git命令]
categories: [Git]
---

### Hexo简介
hexo是一款基于Node.js的静态博客框架，官网地址：https://hexo.io/zh-cn/

### 安装环境

#### 安装Git

程序猿应该基本都会用git了，我在这里就不详细介绍了。

- 建立新的Repository，仓库名为【your_user_name.github.io】
- 后续想要把网站部署到Github上，需要在【your_user_name.github.io】此仓库下的Setting配置中添加一个Deploy keys

Generating a new SSH key可以参考：

- https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/

#### 安装NodeJS

- 因为我是Mac笔记本，所以可以使用homebrew安装
- 也可以下载并安装NodeJS，官网地址：https://nodejs.org/


#### 安装Hexo过程

```
# 使用npm安装Hexo
$ npm install -g hexo-cli

# 初始化Hexo
$ hexo init

# 复制一个markdown文章到{Hexo}/source/_posts目录下
$ cp /Users/ben/Linux常用命令.md {Hexo}/source/_posts/Linux常用命令.md

# 生成静态页面
$ hexo generate

# 启动本地服务，浏览器访问：http://localhost:4000
$ hexo server

# 配置Hexo的_config.yml
deploy:
     type: git
     repo:https://github.com/birdben/birdben.github.io.git
     branch:master

# 要提交到Github上需要安装hexo-deployer-git插件 
$ npm install hexo-deployer-git --save

# 部署网站到Github上
$ hexo deploy

# 访问博客地址：http://birdben.github.io/
# 大功告成!!
```

### 部署步骤

每次部署的步骤，可按以下三步来进行。

- hexo clean
- hexo generate
- hexo deploy

### 更换主题
```
# 从Github下载主题
$ git clone https://github.com/A-limon/pacman.git {Hexo}/themes/pacman

# 修改_config.yml中的theme配置
theme : pacman

# 重新生成静态页面
$ hexo generate

# 启动本地服务，浏览器访问：http://localhost:4000
$ hexo server
```

### 建站目录

```
# 初始化网站目录
$ hexo init <folder>
$ cd <folder>
$ npm install

# 执行完npm install命令后，会生成如下的目录结构
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes

_config.yml : 网站的配置信息
package.json : 应用程序的信息。EJS, Stylus 和 Markdown renderer 已默认安装
scaffolds	: 模版文件夹。当新建文章时，Hexo 会根据 scaffold 来建立文件。
source	: 资源文件夹是存放用户资源的地方。除 _posts 文件夹之外，开头命名为 _ (下划线)的文件 / 文件夹和隐藏的文件将会被忽略。Markdown 和 HTML 文件会被解析并放到 public 文件夹，而其他文件会被拷贝过去。
themes	: 主题 文件夹。Hexo 会根据主题来生成静态页面。
```

### 配置

网站的具体配置，请参考官网：https://hexo.io/zh-cn/docs/configuration.html

### 命令
```
# init : 新建一个网站。如果没有设置folder，Hexo默认在目前的文件夹建立网站。
$ hexo init [folder]

# new : 新建一篇文章。如果没有设置 layout 的话，默认使用 _config.yml 中的 default_layout 参数代替。如果标题包含空格的话，请使用引号括起来。
$ hexo new [layout] <title>

# generate : 生成静态文件。
参数：
-d, --deploy : 文件生成后立即部署网站
-w, --watch : 监视文件变动
$ hexo generate

# publish : 发表草稿。
$ hexo publish [layout] <filename>

# server : 启动服务器。默认情况下，访问网址为： http://localhost:4000/。
$ hexo server
参数：
-p, --port : 重设端口
-s, --static : 只使用静态文件
-l, --log : 启动日记记录，使用覆盖记录格式

# deploy : 部署网站。
$ hexo deploy
参数：
-g, --generate : 部署之前预先生成静态文件

# render : 渲染文件。
$ hexo render <file1> [file2] ...
参数：
-o, --output : 设置输出路径

# migrate : 从其他博客系统迁移内容。
$ hexo migrate <type>

# clean : 清除缓存文件 (db.json) 和已生成的静态文件 (public)。
$ hexo clean

# list : 列出网站资料。
$ hexo list <type>

# version : 显示 Hexo 版本。
$ hexo version

# 安全模式，在安全模式下，不会载入插件和脚本。当您在安装新插件遭遇问题时，可以尝试以安全模式重新执行。
$ hexo --safe

# 调试模式，在终端中显示调试信息并记录到 debug.log。当您碰到问题时，可以尝试用调试模式重新执行一次，并 提交调试信息到 GitHub。
$ hexo --debug

# 简洁模式，隐藏终端信息。
$ hexo --silent

# 自定义配置文件的路径，执行后将不再使用 _config.yml。
$ hexo --config custom.yml

# 显示草稿，显示 source/_drafts 文件夹中的草稿文章。
$ hexo --draft

# 自定义 CWD，自定义当前工作目录（Current working directory）的路径。
$ hexo --cwd /path/to/cwd
```

参考文章：

- https://hexo.io/zh-cn/docs/
- http://www.cnblogs.com/zhcncn/p/4097881.html
- http://www.jianshu.com/p/465830080ea9
- http://opiece.me/2015/04/09/hexo-guide/
- http://www.jianshu.com/p/b7886271e21a