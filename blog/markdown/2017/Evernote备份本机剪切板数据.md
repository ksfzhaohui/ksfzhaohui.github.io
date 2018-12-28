**前言**  
最近同学推荐了一款叫Ditto的软件，用来记录用户的剪贴板数据，包括：文字，图片，文件路径；windows系统本身只能保留最近的一次的剪贴板数据，所以有时候这个功能还是挺有用的；唯一不足的就是不能多端同步，因为一直用印象笔记，所以打算用印象笔记来备份本机剪切板数据，而且印象笔记也提供了强大的搜索功能。

**准备**  
1.申请印象笔记 API Key  
印象笔记本身提供了对外的api接口，我们可以申请API Key，地址：[https://dev.yinxiang.com/doc/](https://dev.yinxiang.com/doc/)，获取API Key即可，如下图：

![](https://static.oschina.net/uploads/space/2017/1029/210746_iEY5_159239.png)

其中要注意的是应用的权限：基本权限和完全权限；基本权限包括创建笔记，列出笔记等；完全权限包括更新删除等功能。

申请完之后会获取一封邮件，如下图所示：

![](https://static.oschina.net/uploads/space/2017/1029/210811_u1yf_159239.png)

主要信息是API Key相关信息，以及告诉我们沙箱环境已经激活，生产环境还没有激活，占时可以在沙箱环境进行测试，并且沙箱环境需要重新创建帐号。

2.evernote-sdk下载  
evernote提供了主流语言的SDK，java sdk对应的地址：[https://github.com/evernote/evernote-sdk-java](https://github.com/evernote/evernote-sdk-java)  
src是sdk的源码，sample是相关demo，可以导入sample进行本地测试

3.用OAuth对印象笔记云 API进行认证  
基于OAuth的认证流程由四部分组成:

```
生成一个临时的Token
请求用户认证
取回 Access Token
接下来的步骤，访问API
```

可以直接将sample中的oauth项目直接导入到Eclipse中，部署到tomcat中，直接访问：[http://localhost:8080/EDAMWebTest/](http://localhost:8080/EDAMWebTest/)  
对应的四个组成部分，界面中也有四个Action：

```
Actions
Get OAuth Request Token from Provider
Send user to get authorization
Get OAuth Access Token from Provider
List notebooks in account
```

分别点击，最终为了获取Access Token，会在页面显示User access token:xxxx

4.简单测试  
有了access token，就可以用sample中的client进行简单的沙箱测试，client提供了EDAMDemo类，需要的AUTH_TOKEN就是刚刚获取的access token，复制进去就可以在沙箱环境(SANDBOX)进行简单的测试了。

**收集剪贴板**  
java提供了类ClipboardOwner用来监听剪贴板数据的变动，剪贴板数据变动，就能直接获取剪贴板的数据，这样就简单了，直接将获取到的剪贴板数据通过Evernote Api将数据同步到印象笔记，部分代码如下：

```
if (clipboard.isDataFlavorAvailable(DataFlavor.stringFlavor)) {
    String text = (String) clipboard.getData(DataFlavor.stringFlavor);
    evernoteApi.createNoteText(text);
    clipboard.setContents(new StringSelection(text), this);
} else if (clipboard.isDataFlavorAvailable(DataFlavor.imageFlavor)) {
    final BufferedImage image = (BufferedImage) clipboard.getData(DataFlavor.imageFlavor);
    ByteArrayOutputStream out = new ByteArrayOutputStream();
    ImageIO.write(image, "png", out);
    Transferable trans = new Transferable() {
               ......
    };
    evernoteApi.createNoteImage("IMAGE:" + new Date(), out.toByteArray());
    clipboard.setContents(trans, this);
} else if (clipboard.isDataFlavorAvailable(DataFlavor.javaFileListFlavor)) {
    @SuppressWarnings("unchecked")
    List<File> array = (List<File>) clipboard.getData(DataFlavor.javaFileListFlavor);
    for (File file : array) {
        evernoteApi.createNoteText(file.getPath());
    }
    clipboard.setContents(contents, this);
} else {
    logger.info("未知的类型");
}
```

代码中主要对三种剪贴板数据类型进行了处理，分别是：DataFlavor.stringFlavor，DataFlavor.imageFlavor和DataFlavor.javaFileListFlavor；对应的是文本，图片和文件，文件只同步了文件的具体路径。

**请求激活生产环境**  
访问地址：[https://dev.yinxiang.com/support/](https://dev.yinxiang.com/support/)，点击“激活API Key”；要求填写具体信息，尽量详细点，然后提示你在2-3个工作日给你激活。

代码做简单修改就可以直接在生产环境运行了，主要修改部分代码：

```
EvernoteApi.Sandbox.class改成EvernoteApi.Yinxiang.class
EvernoteService.Sandbox改成EvernoteService.YINXIANG
```

改完之后接下来和在沙箱环境是类似的，也需要先进行授权，然后获取access token，为了方便，分别提供了在代码中提供了两个bat文件，用来处理这两步，针对每个用户只需要获取一次就可以了。

具体代码地址：  
gitee:[https://gitee.com/OutOfMemory/Clipboard](https://gitee.com/OutOfMemory/Clipboard)  
github:[https://github.com/ksfzhaohui/Clipboard](https://github.com/ksfzhaohui/Clipboard)

**运行效果**  
最后的运行效果，每次执行复制操作，在印象笔记里面就会出现复制的内容：

![](https://static.oschina.net/uploads/space/2017/1029/211057_tuVo_159239.png)