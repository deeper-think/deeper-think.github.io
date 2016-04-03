---
layout: post
title: Centos6.5环境本地安装jekyll环境 
category: 系统及服务
tags: [jekyll]
keywords: jekyll环境,ruby,nodejs
description: 
---

> 我思故我在 -- 笛卡尔

早就想着将博客从cnblog迁到git pages上，一直懒着。趁着周末下雨只能在家窝着，就想把这件事情给做了，git一直都有在使用,对git命令也还算熟悉，所以创建git账号以及建立blog项目都很顺利，借用闫肃同学开源的个人blog代码，很顺利的个人博客就上线了。

接着在本地搭建 -- [jekyll环境](http://jekyllrb.com/), 以前没有用过jekyll，也没有接触过ruby，不过从网上的教程介绍很简单，整个过程敲几个命令就OK了，简直就是小鲜肉，咬上一口爽口又多汁，实际上是一块大骨头，猛地咬上一口，牙齿碎了一地(可能运气不好，各种问题都被碰到了)。

下面总结本地搭建jekyll环境的整个过程及碰到的问题和处理办法，希望能够帮助想要在本地搭建jekyll的人，不用在针对每个问题在Internet上乱搜一通。

## 安装并升级ruby
Centos 6.5用自带的yum源安装ruby为ruby 1.8.7，安装jekyll需要1.9.2以上ruby，因此需要手动编译升级ruby。国内最好用的ruby源应该是 -- [淘宝ruby源](http://ruby.taobao.org/mirrors/ruby/)(淘宝真是一家伟大的公司，为天朝互联网行业能有这样的公司而骄傲)，从ruby上下载 -- [ruby-2.0.0-rc2](http://ruby.taobao.org/mirrors/ruby/2.0/ruby-2.0.0-rc2.tar.gz)源码(当然你也可以根据自己的口味来选择其他的版本，不过根据以往经验，软件包升级时版本号跨度越大遇到的问题越多)，然后configure && make && make install ，这个过程很顺利，孙然 make &&makeinstall的过程中有输出几条failed日志，最后还是安装成功了，如下图所示：

![ruby 升级安装成功](http://7u2rbh.com1.z0.glb.clouddn.com/ruby2.0.png)

重启机器后，ruby -v命令口可以发现本机上ruby版本已经变为：ruby 2.0.0dev。心情很愉悦，以为前戏已经做足了，很快就可以进入下一个环节了，其实不然这都是假象。

## 安装jekyll
上一步ruby已经升级到2.0(其他自认为是)，迫不及待欢快的敲下 gem install jekyll命令，以为马上就是*******(呵呵呵，写作风格得注意点了，担心被天朝的网监查封博客)， 其实不然，这里碰到了第一个问题：

![gem install jekyll:openssl error](http://7u2rbh.com1.z0.glb.clouddn.com/openssl-error.png)

产生该错误的原因是编译ruby时本地环境没有安装openssl-devel，导致前面安装的ruby2.0确实了openssl模块，解决办法是：
1.用自带yum源安装openssl-devel包；
2.进入ruby2.0源码目录：ruby-2.0.0-rc2/ext/openssl，然后执行ruby extconf.rb && make ，这里碰到了第二个问题，ruby openssl模块编译失败，如下所示：

![ruby openssl模块编译失败:error: ‘EC_GROUP_new_curve_GF2m’ undeclared](http://7u2rbh.com1.z0.glb.clouddn.com/openssl complie failed.png)

编译失败的原因是ruby2.0源码上的一个bug，社区已经有相关patch提交，了解具体的bug信息可以参考 -- [ruby Bug#843](https://bugs.ruby-lang.org/issues/8384), 解决的方法是修改源码文件ext/openssl/ossl_pkey_ec.c，具体是：

![ruby 源码修改](http://7u2rbh.com1.z0.glb.clouddn.com/ruby-ssl-sorc-edit.png)

当然你也可以下载已经合入patch的ruby源码重新编译安装，前戏再重新来一次(不过时间就是金钱~~我的朋友，应该利用宝贵的时间来寻找不同的体验)。
OK，按照Bug#843的说明对本地ruby源码文件进行修改，重新make && make install，这次终于成功安装好了ruby openssl模块：

![ruby openssl install success](http://7u2rbh.com1.z0.glb.clouddn.com/ruby-openssl-insuss.png)

到此终于把ruby升级到2.0并且解决了openssl未安装的问题，接下来小心翼翼的再次执行：gem install jekyll，经过漫长的十多分钟的等待后，终于开始输出jekyll安装过程的日志(差点就按CTRL+C了)，最后jekyll终于安装成功：

![jekyll安装成功](http://7u2rbh.com1.z0.glb.clouddn.com/jekyll-install-success.png)

但是，此时执行jekyll 命令会报错，因为本地缺少JavaScript runtime环境，下一步通过安装nodejs来解决这个问题。

## 安装nodejs
首先是到nodejs官网下载最新的 -- [源码](http://nodejs.org/dist/v0.10.35/node-v0.10.35.tar.gz), 解压后configure && make ，编译过程中又报错：

![nodejs编译报错](http://7u2rbh.com1.z0.glb.clouddn.com/nodejs-complie-error.png)

这次报错信息中一个很明显的字符 g++，检查发现环境中g++包果然没有安装，安装g++ 包后重新 make && make install (虚拟机中编译源码包好慢，真怀念前公司个人实验服务器上编译代码的顺畅感觉)，nodejs 安装成功。

## 本地运行jekyll server

最后进入自己本地的blog项目目录，运行 jekyll server，如果一切正常可以看到：Configuration file: /home/deeper-think.github.io/_config.yml Server address: http://0.0.0.0:4000/ 等相关的日志输出，打开本机浏览器输入：http://127.0.0.1:4000/ 可以看到本地博客服务器已经可以正常访问：

![blog 页面](http://7u2rbh.com1.z0.glb.clouddn.com/blog.jpg)

终于大功告成，晚上可以愉快的陪妹子看电影了。


enjoy the life !!!
