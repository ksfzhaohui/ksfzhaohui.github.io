**系列文章**

[CentOS安装Docker](https://my.oschina.net/OutOfMemory/blog/1528285)

[Docker镜像与仓库](https://my.oschina.net/OutOfMemory/blog/1542548)

**什么的是Docker镜像**  
Docker镜像是由文件系统叠加而成，最底层是一个引导文件系统，即bootfs；Docker镜像第二层是root文件系统rootfs，位于引导文件之上，  
可以是一种或多种操作系统；Docker这样的文件系统被称为镜像，一个镜像可以放在另一个镜像的顶部，下面的镜像称为父镜像，最底层的  
镜像称为基础镜像(base image)；当一个镜像启动容器时，Docker会在该镜像的最顶层加载一个读写文件系统，想在Docker中运行的程序就在  
这个读写层中执行。

Docker文件系统层：  
可写容器  
镜像加入Apache  
基础镜像(Ubuntu)  
引导文件系统

**镜像拉取与列出**

```
[root@bogon ~]# docker pull ubuntu
```

pull命令将从Docker hub下载ubuntu镜像，这里没有指定标签(tag)将会自动下载latest标签，即ubuntu:latest；也可以手动指定如ubuntu:12.04

列出刚刚下载的镜像

```
[root@bogon ~]# docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
docker.io/ubuntu        latest              ccc7a11d65b1        3 weeks ago         120.1 MB

```

REPOSITORY中出现的docker.io是官方提供的仓库，具体镜像，标签，仓库，Docker hub之间的关系：  
镜像(image)存放在仓库(repository)中，一个仓库可以存放多个镜像，多个镜像之间通过标签(tag)来区分；  
仓库存放在Registry中，默认的Registry是由Docker官方提供的公共Registry服务，即Docker hub；

注：docker.io是官方提供的仓库，可以有自己的仓库，构建自己的镜像，存放在自己的仓库中；  
Docker Registry代码是开源的，可以运行自己的私有Registry，就行搭建Maven私有仓库一样。

**构建镜像**  
构建镜像的两种方法：  
s1.使用docker commit命令；  
s2.使用docker build命令和Dockerfile文件。  
注：一般来说，我们并不是真正的创建新镜像，而是基于一个已有的基础镜像，如ubuntu等，来构建镜像。

1.创建Docker Hub帐号  
构建镜像中很重要的一环就是如何共享和发布镜像，可以将镜像推送到Docker hub或者私有的Registry中，所以需要一个Docker hub帐号，可以到官网(https://hub.docker.com)注册。

```
[root@bogon static_web]# docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username (ksfzhaohui): ksfzhaohui
Password: 
Login Succeeded
```

使用命令docker login登录到Docker Hub，也可以使用命令docker logout从Registry中退出。

```
[root@bogon static_web]# cd ~/.docker/
[root@bogon .docker]# ll
total 4
-rw-------. 1 root root 103 Sep  3 03:03 config.json
```

用户的个人认证信息保存在~/.docker/config.json下。

2.Docker的commit命令创建镜像  
具体步骤：  
首先使用镜像ubuntu创建一个新容器；  
然后在容器中安装Apache；  
最后使用命令exit从容器中退出，运行commit命令

```
[root@bogon static_web]# docker run -i -t ubuntu /bin/bash
root@ae65f22a003a:/# apt-get -yqq update
root@ae65f22a003a:/# apt-get -y install apache2
Reading package lists... Done
......
root@ae65f22a003a:/# exit
exit
[root@bogon static_web]# docker ps -a
CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS                        PORTS                          NAMES
ae65f22a003a        ubuntu                  "/bin/bash"              18 minutes ago      Exited (100) 17 seconds ago                                  stoic_khorana
[root@bogon static_web]# docker commit ae65f22a003a ksfzhaohui/apache2
sha256:753720533756f8dfc114785434b4a79e5e349cf714d4d59c06844cea8083e38f
[root@bogon static_web]# docker images
REPOSITORY              TAG                 IMAGE ID            CREATED              SIZE
ksfzhaohui/apache2      latest              753720533756        About a minute ago   181.5 MB
docker.io/ubuntu        latest              ccc7a11d65b1        3 weeks ago          120.1 MB

```

commit之后发现images中多了一个ksfzhaohui/apache2镜像

3.用Dockerfile构建镜像  
Dockerfile使用基本的基于DSL(Domain Specific Language)语法的指令来构建一个Docker镜像，推荐使用Dockerfile来代替docker commit方式，因为Dockerfile构建镜像更具备可重复性、透明性和幂等性；  
一旦有了Dockerfile，就可以使用docker build命令来构建一个镜像

```
[root@bogon ~]# mkdir static_web
[root@bogon ~]# cd static_web/
[root@bogon static_web]# touch Dockerfile
[root@bogon static_web]# vi Dockerfile
#Version:0.0.1
FROM ubuntu
MAINTAINER zhaohui "ksfzhaohui@126.com"
RUN apt-get update && apt-get install -y nginx
EXPOSE 80
 
[root@bogon static_web]# docker build -t="ksfzhaohui/static_web" .
Sending build context to Docker daemon 2.048 kB
Step 1 : FROM ubuntu
---> ccc7a11d65b1
Step 2 : MAINTAINER zhaohui "ksfzhaohui@126.com"
---> Running in 13c07b334037
---> cfab5363f92d
Removing intermediate container 13c07b334037
Step 3 : RUN apt-get update && apt-get install -y nginx
......
Step 4 : EXPOSE 80
---> Running in c4981c69d9bb
---> c611fa05de5c
Removing intermediate container c4981c69d9bb
Successfully built c611fa05de5c
```

首先创建了一个文件夹static_web，然后在文件夹下创建了Dockerfile文件，写入指令；  
Dockerfile由一系列的指令和参数组成，每条指令都必须大写字母，且后面要跟一个参数；指令会按顺序从上到下执行，所有需要合理安排指令的顺序。

从以上docker build的日志中可以大体了解按照如下流程执行Dockerfile指令：  
s1.Docker从基础镜像运行一个容器；  
s2.执行一条指令，对容器做出修改；  
s3.执行类似docker commit的操作，提交一个新的镜像层；  
s4.Docker基于刚提交的镜像运行一个新容器；  
s5.执行下一条指令，直到所有指令执行完成。

Dockerfile文件指令说明：  
FROM指令：指定一个已经存在的镜像，后续指令将基于该镜像运行(基础镜像)；  
MAINTAINER指令：告诉该镜像的作者是谁，以及作者的电子邮件；  
RUN指令：在当前镜像中运行指定的命令，这里安装了nginx；  
EXPOSE指令：告诉docker该容器内的应用程序将会使用容器的指定端口；  
注：并不意味着可以自动访问运行的端口(这里是80)，出于安全docker并不会自动打开端口，而是需要用户在使用docker run运行容器时来指定需要打开哪些端口；当然还有其他的Dockerfile指令，此处没有详细介绍。

最后通过docker build指令构建镜像，-t指定了镜像的名称，格式为镜像名:标签，没有指定标签默认是latest

```
[root@bogon ~]# docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
ksfzhaohui/apache2      latest              753720533756        31 minutes ago      181.5 MB
ksfzhaohui/static_web   latest              aa2682ccd397        3 hours ago         215.7 MB
docker.io/ubuntu        latest              ccc7a11d65b1        3 weeks ago         120.1 MB

```

查看镜像列表，ksfzhaohui/static_web镜像已经生成，TAG为latest。

**用构建的新镜像启动容器**

```
[root@bogon static_web]# docker run -d -p 192.168.194.131:8080:80 --name static_web_container ksfzhaohui/static_web \nginx -g "daemon off;"
4ed2cb9166d63c146795a2d28c8f63166d7b1febb846ba0b077b1a489618f304
```

s1.基于构建的镜像ksfzhaohui/static\_web，启动了一个名为static\_web_container的容器；  
s2.-d选项告诉Docker容器以分离的方式在后台运行，也指定了在容器中运行的命令：  
      nginx -g “daemon off;”将以前台运行的方式启动nginx，来作为web服务器；  
s3.-p选项来控制Docker在运行是应该公开哪些端口给外部，192.168.194.131:8080:80将80端口映射到宿主机192.168.194.131的8080端口  
s4.打开浏览器访问[http://192.168.194.131:8080/](http://192.168.194.131:8080/)，可以正常访问nginx

```
Welcome to nginx!
 
If you see this page, the nginx web server is successfully installed and working. Further configuration is required.
 
For online documentation and support please refer to nginx.org.
Commercial support is available at nginx.com.
 
Thank you for using nginx.
```

**将镜像推送到Docker Hub**

```
[root@bogon static_web]# docker push ksfzhaohui/static_web
The push refers to a repository [docker.io/ksfzhaohui/static_web]
621b85d7c682: Pushed
a09947e71dc0: Mounted from library/ubuntu
9c42c2077cde: Mounted from library/ubuntu
625c7a2a783b: Mounted from library/ubuntu
25e0901a71b8: Mounted from library/ubuntu
8aa4fcad5eeb: Mounted from library/ubuntu
latest: digest: sha256:44d55f468b1e8c1cb0a1518289dde38f18c5f8676680cb626661268fdc24e764 size: 1569

```

推送成功，可以打开[https://hub.docker.com](https://hub.docker.com)查看仓库，可以看到如下仓库：  
![](https://static.oschina.net/uploads/space/2017/0924/181514_uttV_159239.png)

**自动构建**  
Docker Hub允许我们定义自动构建(Automated Builds)，为了使用自动构建只需将Github或者BitBucket中含有Dockerfile文件的仓库连接到Docker Hub即可，向这个代码仓库推送代码时，将会触发一次镜像构建活动并创建一个新镜像。

具体步骤：  
s1.打开https://hub.docker.com，Create->Create Automated Builds，选择github进行连接，要求进行授权，点击授权即可；  
s2.github上创建仓库docker，上传一个Dockerfile文件，具体内容可以使用上面创建的Dockerfile文件；  
s3.Create->Create Automated Builds->Create Auto-build Github，然后选择刚刚在github上创建的docker，点击create即可；  
s4.然后就可以看到创建的镜像，如下所示：

![](https://static.oschina.net/uploads/space/2017/0924/181537_hc2q_159239.png)

s5.可以手动触发生成镜像，具体如下：  
![](https://static.oschina.net/uploads/space/2017/0924/181644_RI2J_159239.png)

s6.通过更新Dockerfile文件，然后上传到github，自动触发生成新的镜像；可以简单的修改一下version，然后提交到github，Docker hub马上能获取到最新的Dockerfile，并且重新构建镜像。

**删除镜像**

```
[root@bogon static_web]# docker rmi 753720533756
```

docker rmi 指定镜像名称和id都可以，也支持同时删除多个镜像，通过空格分隔就行。

**总结**  
本文是在看”第一本Docker书”书实战之后做的一些笔记，主要介绍了Docker镜像以及生成，上传和使用镜像等，接下来会进行更加深入的了解。

文章参考：第一本Docker书

**个人博客：[codingo.xyz](http://codingo.xyz/)**