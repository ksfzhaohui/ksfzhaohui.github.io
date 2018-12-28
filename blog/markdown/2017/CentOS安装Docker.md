**系列文章**

[CentOS安装Docker](https://my.oschina.net/OutOfMemory/blog/1528285)

[Docker镜像与仓库](https://my.oschina.net/OutOfMemory/blog/1542548)

**前提条件** 

安装docker有以下前提条件：

1.运行64位CPU架构的计算机

2.运行Liun下3.8或更高版本内核

3.内核必须支持一种适合的存储驱动(storage driver),例如：Device Manager,AUFS,vfs等

**检查前提条件** 

1.检查系统位数

```
[root@bogon ~]# getconf LONG_BIT
64

```

2.检查内核版本

```
[root@bogon ~]# uname -a
Linux bogon 3.10.0-327.el7.x86_64 #1 SMP Thu Nov 19 22:10:57 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux

```

内核版本是3.10，如果centos是6.5版本，内核版本默认是2.6，可以通过以下命令升级到最新内核：

2.1.导入public key

```
[root@bogon ~]# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org

```

2.2.安装ELRepo到CentOS-6.5中

```
[root@bogon ~]# rpm -Uvh http://www.elrepo.org/elrepo-release-6-8.el6.elrepo.noarch.rpm

```

2.3.安装kernel-lt

```
[root@bogon ~]# yum -y --enablerepo=elrepo-kernel install kernel-lt

```

3.检查Device Manager 使用Device Manager最为Docker的存储驱动，为Docker提供存储能力

```
[root@bogon ~]# ls -l /sys/class/misc/device-mapper
lrwxrwxrwx. 1 root root 0 Sep  2 04:39 /sys/class/misc/device-mapper -> ../../devices/virtual/misc/device-mapper

```

可以发现已经安装了Device Manager，如果没有安装可以使用以下命令安装：

```
[root@bogon ~]# yum install -y device-mapper

```

**安装Docker** 

1.centos7可以直接使用命令

```
[root@bogon ~]# yum install docker

```

2.centos6.5可以使用命令

```
[root@bogon ~]# rpm -Uvh http://download.fedoraproject.org/pub/epel/6/i386/epel-release-6-8.noarch.rpm
[root@bogon ~]# yum -y install docker-io

```

3.查看 Docker 是否安装成功

```
[root@bogon ~]# docker version
Client:
 Version:         1.12.6
 API version:     1.24
 Package version: docker-1.12.6-32.git88a4867.el7.centos.x86_64
 Go version:      go1.7.4
 Git commit:      88a4867/1.12.6
 Built:           Mon Jul  3 16:02:02 2017
 OS/Arch:         linux/amd64

Server:
 Version:         1.12.6
 API version:     1.24
 Package version: docker-1.12.6-32.git88a4867.el7.centos.x86_64
 Go version:      go1.7.4
 Git commit:      88a4867/1.12.6
 Built:           Mon Jul  3 16:02:02 2017
 OS/Arch:         linux/amd64

```

4.停止和启动Docker

```
[root@bogon ~]# service docker start
Redirecting to /bin/systemctl start  docker.service

```

```
[root@bogon ~]# service docker stop
Redirecting to /bin/systemctl stop  docker.service

```

5.查看 Docker 信息

```
[root@bogon ~]# docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 1.12.6
Storage Driver: devicemapper
 Pool Name: docker-253:0-67313116-pool
 Pool Blocksize: 65.54 kB
 Base Device Size: 10.74 GB
 Backing Filesystem: xfs
 Data file: /dev/loop0
 Metadata file: /dev/loop1
 Data Space Used: 11.8 MB
 ......

```

返回所有容器和镜像的数量、Docker使用的执行驱动和存储驱动，以及Docker的基本配置

**启动容器** 

使用如下命令docker run启动一个容器：

```
[root@bogon ~]# docker run -i -t ubuntu /bin/bash
Unable to find image 'ubuntu:latest' locally
Trying to pull repository docker.io/library/ubuntu ... 
latest: Pulling from docker.io/library/ubuntu
d5c6f90da05d: Pull complete 
1300883d87d5: Pull complete 
c220aa3cfc1b: Pull complete 
2e9398f099dc: Pull complete 
dc27a084064f: Pull complete 
Digest: sha256:34471448724419596ca4e890496d375801de21b0e67b81a77fd6155ce001edad
root@7a15624dac7d:/#

```

1.执行docker run命令，并指定了-i和-t两个参数，分别表示： -i：指定了标准输入(stdin) -t：为创建的容器分配一个伪tty终端 通过这两个参数新创建的容器可以提供一个交互式的shell

2.接下来的ubuntu是一个镜像的名称，表示docker基于ubuntu镜像来创建容器；这里的ubuntu镜像又被 称为“基础镜像”（类似的fedora、debian、centos等）；在选定的基础镜像上构建其他镜像。 从日志的输出可以看到，首先Docker会先检查本地是否存在ubuntu镜像，如果没有Docker会连接Docker Hub Registry， 查看Docker Hub是否有该镜像，一旦找到就会下载镜像到本地，然后Docker会用这个镜像创建一个新容器。

3.最后指定了 /bin/bash命令，会启动一个Base shell；当容器创建完成，就会执行/bin/bash命令

**使用容器** 

容器正常启动之后，会进入Bash shell，可以在其中像正常使用ubuntu一样，比如：

1.容器的主机名

```
root@7a15624dac7d:/# hostname
7a15624dac7d

```

2.显示系统中各个进程的资源占用状况

```
root@7a15624dac7d:/# top

top - 15:06:36 up  5:53,  0 users,  load average: 0.01, 0.07, 0.06
Tasks:   2 total,   1 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  0.7 sy,  0.0 ni, 99.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  1001332 total,   205456 free,   171760 used,   624116 buff/cache
KiB Swap:  2097148 total,  2097148 free,        0 used.   623364 avail Mem 

   PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                                                         
    11 root      20   0   36660   1720   1256 R  0.3  0.2   0:00.01 top                                                             
     1 root      20   0   18232   1988   1512 S  0.0  0.2   0:00.06 bash

```

3.安装软件 安装vim软件

```
root@7a15624dac7d:/# apt-get update && apt-get install vim
Get:1 http://security.ubuntu.com/ubuntu xenial-security InRelease [102 kB]
Get:2 http://archive.ubuntu.com/ubuntu xenial InRelease [247 kB]
Get:3 http://security.ubuntu.com/ubuntu xenial-security/universe Sources [47.1 kB]
Get:4 http://security.ubuntu.com/ubuntu xenial-security/main amd64 Packages [441 kB]
Get:5 http://security.ubuntu.com/ubuntu xenial-security/restricted amd64 Packages [12.8 kB]
......

```

**容器退出和重启** 

1.使用命令exit命令退出

```
root@7a15624dac7d:/# exit
exit
[root@bogon ~]#

```

2.列出所有的容器 docker ps列出所有正在运行的容器

```
[root@bogon ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES

```

因为刚刚的容器已经退出，已经不在运行

docker ps -a列出所有容器(包括正在运行和不在运行的)

```
[root@bogon ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                       PORTS               NAMES
7a15624dac7d        ubuntu              "/bin/bash"         About an hour ago Exited (127) 2 minutes ago elegant_shirley

```

3.重新启动 使用docker start命令启动，后面跟着要启动容器的ID或者NAMES

```
[root@bogon ~]# docker start 7a15624dac7d
7a15624dac7d

```

重新附着到该容器的会话上，使用docker attach命令

```
[root@bogon ~]# docker attach 7a15624dac7d
root@7a15624dac7d:/#

```

**创建守护式容器** 

1.守护式容器：没有交互式会话，非常适合运行应用程序和服务，大多数情况下都是用守护式方式来运行容器

```
[root@bogon ~]# docker run --name daemon_dave -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"
a7ba9e4f950a5d1d03a9abe7e1816551da4e5e532ec2166dfb7478e9b227f753

```

-d命令表示Docker会将容器放到后台运行，最后在容器的运行命令里面使用了while循环，每秒打印一次hello world

2.docker logs用来获取容器的日志输出

```
[root@bogon ~]# docker logs daemon_dave
hello world
hello world
hello world
hello world

```

3.docker stop用来停止守护容器

```
[root@bogon ~]# docker stop daemon_dave
daemon_dave

```

**删除容器** 

使用命令docker rm

```
[root@bogon ~]# docker rm 7a15624dac7d

```

**总结** 

本文是在看"第一本Docker书"书实战之后做的一些笔记，主要介绍了Docker安装的条件，Docker安装以及容器的简单使用，接下来会进行更加深入的了解。

文章参考：第一本Docker书

**个人博客：[codingo.xyz](http://codingo.xyz/)**