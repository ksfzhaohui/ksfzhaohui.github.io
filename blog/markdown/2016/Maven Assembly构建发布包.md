       maven的出现更近一步的加深了大家对模块化开发的热忱，如何更好的打包，可以让我们工作更加高效，maven提供的Assembly插件能帮你构建一个完整的发布包，下面基于一个实例来发布一个zip包。

gserver项目的名称，包含如下模块：

![](http://static.oschina.net/uploads/space/2016/0601/142109_DGiB_159239.png)

服务:
gserver-gate:网关服务器
gserver-logic:逻辑服务器

公用类:
gserver-core:核心类
gserver-redis:redis缓存管理
gserver-services:逻辑类
gserver-db:mysql数据库管理

打包:
gserver-distribute:打包程序,路径/target目录下

整个项目包含了2个进程以及4个模块，4个公共模块分别给2个进程使用；

整个打包的思路就是：每个模块先各自独立打包，然后通过另外一个模块进行汇总；

我们需要提供了一个专门打包的模块，既项目中出现的gserver-distribute模块，它本身是一个空模块，里面只是提供了Assembly插件需要的xml文件，当然也可能提供一些公共的配置文件。

既然gserver-distribute是一个汇总模块，所有要保证gserver-distribute是最后一个执行模块，如下所示：

```xml
<modules>
	<module>gserver-core</module>
	<module>gserver-redis</module>
	<module>gserver-services</module>
	<module>gserver-gate</module>
	<module>gserver-logic</module>
	<module>gserver-db</module>
	<module>gserver-distribution</module>
</modules>
```

gserver-distribute出现在所有模块的最下面；下面看一下gserver-distribute使用Assembly插件进行打包。

需要提供一个Assembly插件打包的配置文件：assembly.xml，然后再pom.xml进行配置

![](http://static.oschina.net/uploads/space/2016/0601/144157_Tnwu_159239.png)

pom.xml配置使用插件如下:

```xml
<build>
	<plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-assembly-plugin</artifactId>
			<version>2.4</version>
			<configuration>
				<descriptors>
					<descriptor>assembly.xml</descriptor>
				</descriptors>
			</configuration>
			<executions>
				<execution>
					<id>make-assembly</id>
					<phase>package</phase>
					<goals>
						<goal>single</goal>
					</goals>
				</execution>
			</executions>
		</plugin>
	</plugins>
</build>
```

下面再看一下assembly.xml配置文件:

```xml
<assembly>
	<id>bin</id>
	<!-- 最终打包成一个用于发布的zip文件 -->
	<formats>
		<format>zip</format>
	</formats>

	<!-- Adds dependencies to zip package under lib directory -->
	<dependencySets>
		<dependencySet>
			<!-- 不使用项目的artifact，第三方jar不要解压，打包进zip文件的lib目录 -->
			<useProjectArtifact>false</useProjectArtifact>
			<outputDirectory>lib</outputDirectory>
			<unpack>false</unpack>
			<!-- 排除指定的模块 -->
			<excludes>  
                <exclude>com.gserver:gserver-logic</exclude>  
                <exclude>com.gserver:gserver-gate</exclude>  
            </excludes>  
		</dependencySet>
	</dependencySets>

	<fileSets>
		<!-- 把项目的脚本文件目录（ src/main/scripts ）中的启动脚本文件，打包进zip文件的跟目录 -->
		<fileSet>
			<directory>../gserver-logic/shell</directory>
			<outputDirectory></outputDirectory>
			<includes>
				<include>*</include>
			</includes>
		</fileSet>
		<fileSet>
			<directory>../gserver-gate/shell</directory>
			<outputDirectory></outputDirectory>
			<includes>
				<include>*</include>
			</includes>
		</fileSet>
		<fileSet>
			<directory>../gserver-distribution/shell</directory>
			<outputDirectory></outputDirectory>
			<includes>
				<include>*</include>
			</includes>
		</fileSet>

		<fileSet>
			<directory>../gserver-logic/target</directory>
			<outputDirectory></outputDirectory>
			<includes>
				<include>*.jar</include>
			</includes>
		</fileSet>
		<fileSet>
			<directory>../gserver-gate/target</directory>
			<outputDirectory></outputDirectory>
			<includes>
				<include>*.jar</include>
			</includes>
		</fileSet>

		<fileSet>
			<directory>../gserver-logic/logic-config</directory>
			<outputDirectory>logic-config</outputDirectory>
			<includes>
				<include>*</include>
			</includes>
		</fileSet>
		<fileSet>
			<directory>../gserver-gate/gate-config</directory>
			<outputDirectory>gate-config</outputDirectory>
			<includes>
				<include>*</include>
			</includes>
		</fileSet>
	</fileSets>
</assembly>  
```

主要的设置大概就这些，更加详细的：[https://github.com/ksfzhaohui/gserver](https://github.com/ksfzhaohui/gserver)

整个包最后将打成一个zip包，当然还支持其他很多格式，更多细节可以去看Maven Assembly插件，这里简单介绍一下整个流程。

最后的zip包:gserver-distribution-0.0.1-SNAPSHOT-bin.zip,内部如下图所示:  
![](http://static.oschina.net/uploads/space/2016/0601/145208_itmU_159239.png)

好了，这时候你可以直接丢给客户端，测试，策划，让他们一键启服(all-startup.bat),一键停服(all-shutdown.bat)