# 关于博客

简单的说一下这个博客的部署。

这是一个基于[Gitbook](https://www.gitbook.com/)的电子书，通过markdown语法以及Gitbook约定的文档结构，可以很有逻辑的讲内容展示出来。整个博客项目托管在我个人的github：https://github.com/huangwjwork/gitbook ，然后通过github与我的vps部署的webhook联动：每当有push行为，github就会post一条请求到我vps上的webhook，然后触发我的gitbook脚本，实现博客的自动更新。

关于Gitbook，想要了解的可以参考这位大佬的博客：`http://gitbook.zhangjikai.com/`

关于github的webhook的部署，网上的资源很多，我用的是`https://github.com/hustcc/webhookit`，内有中文文档

## 安装Gitbook

### 安装node.js

编译安装或者对应的二进制安装

```shell
yum install nodejs
```

### 安装Gitbook

国内环境建议修改npm源为淘宝镜像站后再安装

```shell
npm install gitbook-cli -g
gitbook -V
```

## 配置gitbook

可以在github上创建一个项目，然后clone到本地，进入项目根目录，执行`gitbook init`，编辑gitbook.json，SUMMARY.md，README.md，以及.gitignore

### book.json

book.json是gitbook的配置文件，包括插件的配置文件，通过插件可以丰富电子书的功能，有兴趣的可以去官方找找，很多很有意思的插件（插件越多js文件越多，我的vps流量计费，所以我的是乞丐版 T T）

贴一下我的book.json

```json
cat book.json 
{
    "title": "huangwjwork's notes",
    "description": "好记性不如烂笔头，记录日常遇到的问题及学习的成果",
    "author": "huangwjwork",
    "output.name": "site",
    "language": "zh-hans",
    "gitbook": "3.2.3",
    "root": ".",
    "links": {
        "sidebar": {
            "Home": "https://huangwj.app"
        }
    },
    "plugins": [
        "github@^2.0.0",
        "edit-link@^2.0.2",
        "anchors@^0.7.1",
        "include-codeblock@^3.0.2",
        "splitter@^0.0.8",
        "tbfed-pagefooter@^0.0.1",
        "expandable-chapters-small@^0.1.7",
        "anchor-navigation-ex@0.1.8"
    ],

    "pluginsConfig": {
        "theme-default": {
            "showLevel": true
        },
        "github": {
            "url": "https://github.com/huangwjwork/gitbook"
        },
        "include-codeblock": {
            "template": "ace",
            "unindent": true,
            "edit": true
        },
        "tbfed-pagefooter": {
            "copyright": "Copyright © huangwjwork 2017",
            "modify_label": "该文件修订时间：",
            "modify_format": "YYYY-MM-DD HH:mm:ss"
        },
        "edit-link": {
            "base": "https://github.com/huangwjwork/gitbook/edit/master",
            "label": "Edit This Page"
        },
        "anchor-navigation-ex": {
            "isRewritePageTitle": false,
            "tocLevel1Icon": "fa fa-hand-o-right",
            "tocLevel2Icon": "fa fa-hand-o-right",
            "tocLevel3Icon": "fa fa-hand-o-right"
        }


    }
}
```

编写完成后在book.json文件目录执行如下命令安装插件

```
gitbook install
```



### SUMMARY.md

概要文件主要存放 GitBook 的文件目录信息，左侧的目录就是根据这个文件来生成的，它通过 [Markdown](http://gitbook.zhangjikai.com/GLOSSARY.html#markdown) 中的列表语法来表示文件的层级关系，下面是一个简单的示例： 

```shell
# Summary

* [Introduction](README.md)

-----
* [个人简历](ABOUT_ME.md)

-----
* [关于博客](ABOUT_BLOG.md)

-----
* [知识库](knowledge.md)
    * [操作系统](OS/os.md)
        * [Linux](OS/linux/linux.md)
        * [windows](OS/win/windows.md)
        * [Unix](OS/unix/unix.md)
```

编写完成后，可以执行init命令让gitbook自动生成上述目录结构

```shell
$ gitbook init
info: create SUMMARY.md
info: initialization is finished
```



### README.md

电子书的主页，可以在book.json中修改

### .gitignore

由于生成电子书时会产生大量的nodejs文件以及_gitbook的电子书文件，建议配置`.gitignore`

```shell
[huangwj@instance-1 ~]$ cat /opt/huangwj/gitbook/.gitignore 
/_book/
/node_modules/
```



## 生成gitbook电子书

主要配置文件编辑完成后，就可以生成gitbook电子书，默认生成html，可以在本地起服务查看，也可以将html拷贝到web服务器下查看

* 本地查看，默认端口4000，可以更改

  ```shell
  $ gitbook serve
  Live reload server started on port: 35729
  Press CTRL+C to quit ...
  
  info: 41 plugins are installed
  info: 15 explicitly listed
  info: loading plugin "github"... OK
  info: loading plugin "edit-link"... OK
  info: loading plugin "anchors"... OK
  info: loading plugin "include-codeblock"... OK
  info: loading plugin "splitter"... OK
  info: loading plugin "tbfed-pagefooter"... OK
  info: loading plugin "expandable-chapters-small"... OK
  info: loading plugin "anchor-navigation-ex"... OK
  info: loading plugin "livereload"... OK
  info: loading plugin "highlight"... OK
  info: loading plugin "search"... OK
  info: loading plugin "lunr"... OK
  info: loading plugin "sharing"... OK
  info: loading plugin "fontsettings"... OK
  info: loading plugin "theme-default"... OK
  info: found 26 pages
  info: found 27 asset files
  warn: "options" property is deprecated, use config.get(key) instead
  info: >> generation finished with success in 5.0s !
  
  Starting server ...
  Serving book on http://localhost:4000
  ```

  



## 部署webhook并与github联动

### 安装webhookit

安装webhookit，并生成默认配置文件，请注意自己的Python环境，调用相应的pip

```shell
pip install webhookit
webhookit_config > /opt/webhook/webhook_for_github.conf 
```

修改配置文件

* ​	如果执行脚本在webhook本机，只需要修改如下两个参数
  * `repo_name/branch_name`修改成自己的项目名称和分支名
  * `SCRIPT`写入自己要执行的脚本

```shell
[huangwj@instance-1 ~]$ cat /opt/webhook/webhook_for_github.conf 
# -*- coding: utf-8 -*-
'''
Created on May-25-18 19:10:16

@author: hustcc/webhookit
'''


# This means:
# When get a webhook request from `repo_name` on branch `branch_name`,
# will exec SCRIPT on servers config in the array.
WEBHOOKIT_CONFIGURE = {
    # a web hook request can trigger multiple servers.
    'gitbook/master': [{
        # if exec shell on local server, keep empty.
        'HOST': '',  # will exec shell on which server.
        'PORT': '',  # ssh port, default is 22.
        'USER': '',  # linux user name
        'PWD': '',  # user password or private key.

        # The webhook shell script path.
        'SCRIPT': '/opt/huangwj/scripts/gitbook_update.sh > /opt/huangwj/scripts/gitbook_update.log'
    }]
}
```

我的脚本

```shell
[huangwj@instance-1 scripts]$ cat gitbook_update.sh 
#!/bin/bash
source /etc/profile
source /home/huangwj/.bash_profile
date
cd /opt/huangwj/gitbook
git pull
gitbook install
gitbook build
```

### 启动webhookit

```shell
webhookit -c /opt/webhook/webhook_for_github.conf -p port
```

启动完成后即可访问localhost:port查看webhook的信息及推送的URL，在github填入URL并配置type为json即可。

## 配置github

项目——setting——webhook——ADD webhook

* payload URL：填写webhookURL
* Content type ：application/json

触发条件可选，我这里选择的是`Just the push event.` 



