**1.环境**

eclipse：Mars.2 Release (4.5.2)  
jdk:1.7

**2.安装gradle插件**

Help->Install new software,输入[http://dist.springsource.com/release/TOOLS/gradle](http://dist.springsource.com/release/TOOLS/gradle) 或者  
Help->Eclipse Marketplace,输入 gradle  
低版本(3.7)的eclipse安卓gradle插件总是报错：

```
An error occurred while collecting items to be installed
session context was:(profile=epp.package.jee, phase=org.eclipse.equinox.internal.p2.engine.phases.Collect, operand=, action=).
Problems downloading artifact: osgi.bundle,org.springsource.ide.eclipse.gradle.core,3.6.3.201411271013-RELEASE.
Error reading signed content:C:\Users\dell\AppData\Local\Temp\signatureFile1990886222537489997.jar
```

可以查看windows->Preferences 添加了Gradle(STS)目录

**3.创建project**  
创建项目T1，在根目录下添加build.gradle文件  
build.gradle：  

```
apply plugin: 'java'
apply plugin: 'eclipse'
archivesBaseName = 'GradleTest'
version = '1.0-SNAPSHOT' 
repositories {
    mavenCentral()
}
jar {
	manifest {
		attributes 'Main-Class': 'com.test.Test'
	}
}
dependencies {
   compile  'org.apache.commons:commons-lang3:3.0'
   compile  'log4j:log4j:1.2.16'

}
```

如下图：  
![](http://static.oschina.net/uploads/space/2016/0321/145158_COBU_159239.png)

接下来在T1上面右击->Configure->Convert to Gradle(STS)Project,转化之后如下图:  
![](http://static.oschina.net/uploads/space/2016/0321/145506_FXwf_159239.png)

T1上面多了一个G，并且依赖的jar也帮我们引入了项目中

**4.打包**  
方式一：

T1右击->Run As->Gradle(STS) Build...  输入:clean build  
![](http://static.oschina.net/uploads/space/2016/0321/145849_2zsA_159239.png)  
Run之后：

```
[sts] -----------------------------------------------------
[sts] Starting Gradle build for the following tasks: 
[sts]      clean
[sts]      build
[sts] -----------------------------------------------------
:clean UP-TO-DATE
:compileJava
:processResources UP-TO-DATE
:classes
:jar
:assemble
:compileTestJava UP-TO-DATE
:processTestResources UP-TO-DATE
:testClasses UP-TO-DATE
:test UP-TO-DATE
:check UP-TO-DATE
:build

BUILD SUCCESSFUL

Total time: 0.155 secs
[sts] -----------------------------------------------------
[sts] Build finished succesfully!
[sts] Time taken: 0 min, 0 sec
[sts] -----------------------------------------------------
```

打包成功，T1目录下面多了一个build文件夹，libs/GradleTest-1.0-SNAPSHOT.jar  
  
方式二:

当然也可以不在eclipse里面进行打包，官网下载gradle:[http://gradle.org/gradle-download/](http://gradle.org/gradle-download/)

当前下载的版本是gradle-2.4-bin.zip，然后设置环境变量:  
GRADLE_HOME=D:\\gradle-2.4  
Path+=%GRADLE_HOME%/bin

下面就可以在命令行中使用:gradle clean build  
![](http://static.oschina.net/uploads/space/2016/0321/150705_M9NY_159239.png)

个人感觉Gradle和maven思想上都差不多，而且很多方面直接继承了maven，可能就是在配置上更加简练一点，以后有机会可以再深入了解一下。