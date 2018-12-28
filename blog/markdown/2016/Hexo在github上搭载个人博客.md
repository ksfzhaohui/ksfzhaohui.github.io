**准备**

1.Git-2.5.1  下载地址:[https://git-scm.com/download/](https://git-scm.com/download/)  
2.node-v4.4.1 下载地址:[https://nodejs.org/en/](https://nodejs.org/en/)

**安装hexo**

在指定文件夹下(E:/hexo)下，右击->Git Bash Here,下面可以开始安装hexo了：  

```
npm install -g hexo
```

因为后面会上传hexo到github，所以要安装一个插件,不然会报: ERROR Deployer not found : github，安装插件  

```
npm install hexo-deployer-git  --save
```

初始化

```
hexo init
```

执行init命令初始化hexo到你指定的目录,也就是当前的E:/hexo目录

下面的命令是要经常使用的：

```
$ hexo g #完整命令为hexo generate,用于生成静态文件
$ hexo s #完整命令为hexo server,用于启动服务器，主要用来本地预览
$ hexo d #完整命令为hexo deploy,用于将本地文件发布到github上
$ hexo n #完整命令为hexo new,用于新建一篇文章
$ hexo clean #用于清空缓存
```

init之后，现在的E:/hexo文件夹下：  
![](http://static.oschina.net/uploads/space/2016/0326/171527_O0b8_159239.png)

source:里面存放的是我们的博客源码  
themes:存放的不同的主题  
_config.yml:主配置文件

生成静态文件：

```
hexo g
```

这时候文件夹下多了一个public文件夹，里面是生成的静态页面文件，如果希望重新生成，可以先执行：

```
hexo clean
```

会将public文件夹删除

**上传github**

到此已经准备好了我们的博客，下面我们可以:  
1.本地测试一下

```
hexo s
```

启动服务器，通过localhost:4000访问，确定没问题了，可以上传到github上了。

2.上传到github上

2.1在上传到github上之前，我们需要在github上new respository，创建respository有固定的写法：  

```
your_user_name.github.io
```

我的用户名是ksfzhaohui,所有我创建的resposity是：[http://ksfzhaohui.github.io/](http://ksfzhaohui.github.io/)

2.2创建好之后我们还需要修改主配置文件_config.yml，打开文件，拉到文件的最尾部：

```
deploy:
  type:
```

此处我们要改成我们的resposity:  

```
deploy:
  type: git
  repository: https://github.com/ksfzhaohui/ksfzhaohui.github.io.git
  branch: master
```

2.3上传hexo

```
hexo d
```

上传之后我们文件夹下面会多一个.deploy_git文件夹，此文件夹和github上面是同步的，

下面就可以访问了，比如我的博客地址：[http://ksfzhaohui.github.io/](http://ksfzhaohui.github.io/)

**写博客**

```
hexo n 博客名称
```

会在source/_posts文件夹下创建一个md文件，可以学习一下[Markdown 语法说明](http://wowubuntu.com/markdown/#p)，写完之后可以按照上面的步骤执行:

```
hexo g  生成静态文件
hexo d  发布
```

**扩展性**

hexo提供了很好的扩展性，包括主题以及各种插件；

[官方提供的各种主题](https://github.com/hexojs/hexo/wiki/Themes)，可以通过_config.yml文件进行相关的配置，此处先不进行说明了