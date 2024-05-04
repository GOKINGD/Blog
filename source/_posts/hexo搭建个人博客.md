---
title: hexo搭建个人博客
date: 2024-05-04 19:00:58
tags: 搭建博客
categories: Others
---
**摘要**：本文主要介绍如何利用hexo框架快速搭建个人博客。
**基本认识**：首先hexo是一个静态博客框架，可以快速生成静态博客页面，从而节省建站的前后端操作时间。通常，需要将生成的静态博客页面部署到服务器上，部署完成后可以利用ngix实现通过访问IP地址来访问博客。若要通过域名访问，则需要在购买域名并进行备案后，添加域名并进行域名解析。若没有服务器，可以利用免费的Github Pages进行部署，这样可以通过访问 用户名+github.io这个域名进行访问。

# Hexo

笔者直接在服务器(假设IP为1.2.3.4)上进行了操作，并将静态页面也部署到了本服务器上的不同位置。
首先下载Hexo
``` bash
$ npm install hexo-cli -g
$ hexo init blog #这里的blog是博客的名字
$ cd blog
$ npm install
``` 

创建项目后，目录结构如图所示：
{% asset_img image-1.png image-1 %}
其中主要关注scaffolds、source、themes目录以及_config.yml文件
其中
scaffolds：模板文件夹。当新建文章时，Hexo 会根据scaffold 来建立文件。Hexo 的模板是指在新建的markdown 文件中默认填充的内容（骨架）
例如：如果修改scaffold/post.md 中的Front-matter 内容，那么每次新建一篇文章时都会包含这个修改
source：用来存放文章的源文件，其中_posts存放文章，而_images是我自己用来存放图片的目录
themes: 用来存放主题
_config.yml: 配置网站的基础参数

# 部署

hexo new 可以快速创建一篇博客，会在/blog/source/_posts/ 目录下创建Post.md文件。
``` bash
$ hexo new "Post" #Post是博客文件名
```
hexo server 启动服务器，用于预览主题。可以理解为部署到本地，其默认端口是4000，即通过 http://localhost:4000 可以本地预览。
``` bash
$ hexo server # hexo s
``` 
hexo generate 用于生成静态文件
``` bash
$ hexo generate #hexo g
```
最终，hexo deploy部署博客，可以理解为最终远程部署上线。
``` bash
$ hexo deploy #hexo d
```

通常在修改配置或者新增博文后，会执行hexo clean清除缓存，然后再执行上述的各种命令
``` bash
$ hexo clean
```

# 部署到Github

部署到github很简单，首先在github创建一个新的仓库，仓库名一定要是用户名.github.io, 然后把部署后的网站push到仓库中就OK了。
可以在仓库的设置页面中的Pages选项中看到网站更详细的配置，注意选择的分支是master还是main分支。
{% asset_img image.png image %}

另外还需要在部署之前修改_config.yml文件，如下所示：
``` yml
deploy:
  type: git
  repo: https://github.com/GOKINGD/gokingd.github.io.git
  branch: master
```
这样，在执行hexo d的时候，就会通过git的方式将hexo生成的静态前端页面一起推送到远端仓库

# 部署到服务器
部署到自己的服务器上原理和部署到github上是类似的。
首先在服务器上创建repo仓库, 可以理解为github上的仓库，用于对生成的前端内容进行版本控制
``` bash
mkdir /www/repo
cd /www/repo
git init --bare hexo.git
```
然后建立网站的根目录hexo，hexo用于存放生成的前端页面内容，repo就是控制hexo目录下的git信息
``` bash
mkdir /www/hexo
```
然后创建Git钩子，用于自动部署
``` bash
vim /www/repo/hexo.git/hooks/post-receive
```
粘贴下面的代码
``` bash
#!/bin/bash
git --work-tree=/www/hexo --git-dir=/www/repo/hexo.git checkout -f
```
上面的git命令的意思可以理解为：将指定仓库（/www/repo/hexo.git）的内容强制恢复到当前分支的状态，并将结果文件放置在指定的工作目录（/www/hexo）中。
可以理解为，在hexo d部署的时候，通过git推送到repo目录下，再将内容恢复到hexo目录下,这样的做法是为了方便部署的时候使用git进行推送。

还需要配置_config.yml
``` yml
deploy:
  type: git
  repo: root@1.2.3.4:/www/repo/hexo.git
  branch: master
```
接下来，为了能够使用ip地址直接访问博客(http://1.2.3.4) ,需要配置nginx

# 配置nginx
安装nginx安装后，需要修改默认网站页面，由于本地环境是ubuntu22.04lts，所以修改/etc/nginx/sites-available/default文件
root 后的地址改为博客网页的地址，也就是之前建立的/www/hexo目录
{% asset_img image-2.png image-2 %}

# 更换博客主题
hexo支持各种主题，可以直接从官方主题库下载：https://hexo.io/themes/
本站使用的是Butterfly主题，首先在themes目录下下载主题
``` bash
git clone -b master https://gitee.com/immyw/hexo-theme-butterfly.git themes/butterfly
npm install hexo-renderer-pug hexo-renderer-stylus --save #安装 pug 以及 stylus 的渲染器
```
最后修改_config.yml 并重新部署即可生效
``` yml
theme: butterfly
```