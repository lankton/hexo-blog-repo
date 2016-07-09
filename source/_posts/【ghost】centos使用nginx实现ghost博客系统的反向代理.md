---
title: 【nginx】centos使用nginx实现ghost博客系统的反向代理
date: 2016-07-09 21:47:22
categories: Lan's tech
tags:
  -  杂学
  -  nginx
---

整个8月份都没有更新博客，一方面是自己在忙一些有的没的，另一方面，也是懒的缘故吧。。。
最近在玩基于nodejs的开源博客系统ghost，用阿里云服务器(centOS)搭建了一个属于自己的个人博客，还是挺有意思的。具体的nodejs + mysql + ghost配置教程，网上有很多，我基本上也是照着配置的，这里就不再详述。
这里主要介绍一下在配置完成之后，如果通过nginx实现反向代理，使得访问ghost时，可以忽略端口号，直接通过ip地址或者域名进行访问。
### 使用nginx之前
在使用nginx之前，如果想要通过浏览器访问到服务器上的ghost，需要对ghost工程根目录下的配置文件config.js中的相关配置进行修改，将其中的ip地址从默认的127.0.0.1换成云服务器的地址。修改完成之后重启ghost，在本地可以通过ip地址+端口号成功访问，如图所示（2368是ghost的默认端口，可以在config.js里修改）：
![这里写图片描述](http://img.blog.csdn.net/20150911163136807)

ok,这样可以成功访问了。但这样存在几个明显的弊端：
1、将阿里云的ip写死在ghost工程的config.js里，如果ip地址发生变化，就要不断修改config.js（多处）。
2、web访问的缺省端口号为80，这种端口号非80的地址，无法直接通过ip地址访问，也就等于无法直接使用域名访问。有人也许会说，那把ghost工程运行的端口号改成80不就可以 了。当然，这样也是可以的，但在工程里改来改去总归不好。
这个时候，如果可以在服务器上部署一个工具，使得用户访问80端口时，可以自动映射到相应的web工程端口（这里为ghost的2368）问题就能解决了。这个工具就是nginx。

### 使用nginx
1、安装nginx:
 yum install nginx (这里的环境为Centos，其它Linux环境可以将yum 替换为apt-get)
2、给nginx添加配置文件：
进入目录:etc/ nginx
![这里写图片描述](http://img.blog.csdn.net/20150911165132583)
其中conf.d目录存放自定义配置文件，打开nginx目录下的nginx.conf文件，可以看到最后一行
![这里写图片描述](http://img.blog.csdn.net/20150911165450568)，
即引用了conf.d目录下的所有配置文件。所以进入conf.d目录，创建新的配置文件ghost.conf,内容如图:
![这里写图片描述](http://img.blog.csdn.net/20150911170252590)
很明显可以看出，这正是设置的80端口到本地2368端口的映射，<font color="#FF0000">**而且由于直接映射的127.0.0.1，所以ghost工程的配置文件config.js也不需要修改了。**</font>
这个时候直接启动nginx会报错，因为同样conf.d目录下的default.conf也是配置 的80端口的映射，会造成重复映射。我的解决办法是将default.conf重命名为default.conf.unenable， 这样其就 不会在nginx.conf中被引用了。


<font color="#32CD32">特别说明：不同的系统环境可以安装的nginx目录结构有所不同，但只要记住一点，你创建的配置 文件能够在nginx.conf中被引用到就可以了。</font>

3、启动nginx：
<font color="#FF0000">如果之前修改了ghost工程目录config.js中的ip，将其恢复成127.0.0.1，当然，也可以在ghost.conf中将127.0.0.1换成服务器的外网ip。</font>
命令：
service nginx start

这个时候如果ghost工程也是开启的，那就可以在本地直接通过ip地址（即 使用缺省端口80） 访问到了。
![这里写图片描述](http://img.blog.csdn.net/20150911172046687)

域名和ip绑定了,也可以直接通过域名访问: [http://lankton.cn](http://lankton.cn)
