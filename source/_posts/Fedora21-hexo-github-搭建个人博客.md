---
title: Fedora21 + hexo + github 搭建个人博客
date: 2016-08-24 23:01:37
categories: [教程]
tags: [linux, hexo, github]
---

 介绍自己的配置过程和在配置的时候遇到的各种问题，该文选用的主题是yilia，如果没有js基础，
 绑定github建议不使用yilia主题，使用yilia主题需要修改代码，建议采用NexT主题，配置
 起来简单快捷。

 对与如何配置NexT主题，本文不做叙述。

 以下内容只针对yilia主题的配置和修改方法，安装和配置不是本文的重点，重点在于从github克隆
 yilia主题后，对yilia的修改使显示正常。

 修改后的yilia将会放在我的github上，如果需要可以直接去下载。

<!-- more -->

## **环境搭建** ##

 git的安装和github账号不在介绍的范围

 - **安装node.js**  

``` bash
$ sudo yum install nodejs.i686
```

 - **安装npm**

``` bash
$ sudo yum install npm.noarch
```

 - **安装hexo**

``` bash
$ sudo npm install hexo-cli -g
```

## **命令的使用**
 - `$ hexo init [folder] `
  
 	新建一个网站。如果没有设置 folder ，Hexo 默认在目前的文件夹建立网站。

 - `$ hexo generate` 或 `$ hexo g`

	生成静态文件。

 - `$ hexo server`
	
	启动服务器。默认情况下，访问网址为： http://localhost:4000/。

 - `$ hexo deploy` 或`$ hexo d`
 
	部署网站。

 - `$ hexo clean`
 
	清除缓存文件 (db.json) 和已生成的静态文件 (public)。

## **搭建**

### **1. 建站**

 以blog目录为例，blog作为博客根目录，所有的操作都在里面进行

``` bash
$ hexo init bolg

$ cd blog
$ npm install
```

### **2. 本地预览**

``` bash
$ hexo g    //生成静态文件
$ hexo s    //启动本地服务
```

 成功后，使用浏览器输入localhost:4000，将出现
 ![fhg1](/img/fhg1.png)

### **3. 更换主题**
``` bash 
mkdir themes/yilia  //如果没有对应的目录，创建
git clone git@github.com:litten/hexo-theme-yilia.git themes/yilia
```

修改blog目录下的config.yml配置文件中的theme属性，将其修改为yilia。
``` bash
theme: yilia
```

### **4. 配置**
Hexo 中有两份主要的配置文件，其名称都是 _config.yml。 其中，一份位于站点根目录下，
主要包含 Hexo 本身的配置；另一份位于主题目录下，这份配置由主题作者提供，
主要用于配置主题相关的选项。

为了描述方便，在以下说明中，将前者称为 站点配置文件， 后者称为 主题配置文件

#### **修改站点配置文件**
修改config.yml中的title、subtitle、author、language属性
``` bash
title: Szp Blog
subtitle: Qt,C,MySQL,TCP/IP,前端
author: szping
language: zh-Hans
```

#### **修改主题配置文件**
``` bash
avatar: "/img/linux.jpg"    //修改头像，图片放在theme/yilia/source/img
animate: false              //关闭动画显示
```

### **5. 绑定github**
 首先，要在自己的github上创建一个仓库，建立与你用户名对应的仓库，
 仓库名必须为【your_user_name.github.io】，固定写法
 比如:github用户名为xiaowang，仓库名应为xiaoming.github.io，不区分大小写
 
 修改站点配置文件的deploy属性，绑定自己的github
 ``` bash
deploy:
    type: git
    repository: git@github.com:Linuxszp/linuxszp.github.io.git
    branch: master
 ```
 对应的属性如果没有，需要自己添加

### **6. 修改yilia，使其本地显示正常**
使用
```bash
$ hexo g
$ hexo s
```
后，打开浏览器输入http://localhost:4000/，该动作每一次修改都会执行一遍，出现如下界面
 ![fhg2](/img/fhg2.png)
发现头像和标签显示不正常，并且鼠标点击小房子也没反应

调试发现以下错误
![fhg3](/img/fhg3.png)

解析头像图片时路径中多了个null，yiliaConfig没有定义，实际上是因为在主题配置文件中
root: 为空引起的，需要修改以下部分的代码
修改../yilia/layout/_partial/after-footer.ejs中的第5～18行代码为：
```bash
<script>
    var yiliaConfig = {
        fancybox: <%=theme.fancybox%>,
        mathjax: <%=theme.mathjax%>,
        animate: <%=theme.animate%>,
        isHome: <%=is_home()%>,
        isPost: <%=is_post()%>,
        isArchive: <%=is_archive()%>,
        isTag: <%=is_tag()%>,
        isCategory: <%=is_category()%>,
        open_in_new: <%=theme.open_in_new%>
    }
</script>
```

修改../yilia/source/js/main.js中的第4～8行为：
```bash
    var loadMobile = function(){
        require(['/js/mobile.js'], function(mobile){
            mobile.init();
            isMobileInit = true;
        });
    }
```

修改../yilia/source/js/main.js中的第11～16行为：
```bash
    var loadPC = function(){
        require(['/js/pc.js'], function(pc){
            pc.init();
            isPCInit = true;
        });
    }
```

修改../yilia/source/js/main.js中的第58行和第75行为：
![fhg4](/img/fhg4.png)

修改../yilia/layout/_partial/left-col.ejs中的5～9行代码为：
```bash
<% if (theme.animate){ %>
    <img lazy-src="<%=theme.avatar%>" class="js-avatar">
<%}else{%>
    <img src="<%=theme.avatar%>" class="js-avatar" style="width: 100%;height: 100%;opacity: 1;">
<%}%>
```

修改../yilia/layout/_partial/mobile-nav.ejs中的9～13行代码为：
```bash
<% if (theme.animate){ %> 
    <img lazy-src="<%=theme.avatar%>" class="js-avatar">
<%}else{%>
    <img src="<%=theme.avatar%>" class="js-avatar" style="width: 100%;height: 100%;opacity:1;">
<%}%>
```

修改../yilia/source/css/_partial/highlight.styl中的第66～69行代码为：
```bash
        .line
            font-size: 15px
    .gist
```
去掉了`height: 15px`可以使显示代码的时候没有滚动条

修改后，重新执行一下，显示正常，点击小房子也有反应了
![fhg5](/img/fhg5.png)



### **7. 部署**
 将博客与GitHub Pages绑定后就可以部署在服务器上了，这里是部署在github，
 还以xiaoming为例
``` bash
$ hexo d
```
如果出现ERROR Deployer not found: git错误，则运行下面命令
```bash
npm install hexo-deployer-git --save
```
执行后，再部署
成功后，就可以使用linuxszp.github.io访问自己的博客了

### **8. 修改yilia，使其网络访问显示正常**
部署后，使用linuxszp.git.io访问，显示不正常
![fhg6](/img/fhg6.png)
点击小房子没反应和标签显示不出来，只显示小圆圈

调试发现一下错误
![fhg7](/img/fhg7.png)
加载的是https，而请求的是一个http一个不安全的脚本，需要修改使yilia兼容https

修改../yilia/layout/_partial/mathjax.ejs中的第18行代码为：
```bash
<script type="text/javascript" src="//cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
```

修改../yilia/layout/_partial/after-footer.ejs中的第18行代码为：
```bash
<%- js('//7.url.cn/edu/jslib/comb/require-2.1.6,jquery-1.9.1.min') %>
```

修改后显示正常
![fhg8](/img/fhg8.png)
