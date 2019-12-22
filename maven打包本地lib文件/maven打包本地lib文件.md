# 记一次maven打包本地lib中jar包记录

java项目使用的maven进行依赖以及打包管理，后续更新到服务器使用docker-compose管理项目中的多个main函数执行数据流分析操作。本次就只说说其中遇到的一个问题以及后续的解决办法。

一开始直接使用maven引入所依赖的jar包即可，后续打包在`pom.xml`使用的如下配置：

```xml
<plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
        <archive>
            <manifest>
                <mainClass>
                    com.duoyi.SparkMinuteCount
                </mainClass>
            </manifest>
            <descriptorRefs>
                <descriptorRef>jar-with-dependencies</descriptorRef>
            </descriptorRefs>
    </configuration>
</plugin>
```

然后执行如下命令即可将项目打包成jar包：（下面两个命令可以合并起来，需要在`pom.xml`文件中增加配置将`assembly:single`添加到`compile`的生命周期中，暂时略过）

```bash
mvn compile
mvn assembly:single
```

运行时执行：

```bash
java -cp jarFileName.jar com.main [args...]
或
java -jar jarFileName.jar
```



## 0x01 问题1：compile失败

### 1. 问题描述

后续引入其他功能时，需要使用到一个手动编译的jar包，maven中央仓库中没有，所以在项目根目录中新建`lib\`目录，将该jar包放进去，在idea中在lib目录使用右键`Add As Library...`将该目录添加到依赖。

开发期间木有问题，最后测试打包时候发现有问题，`compile`期间提示package缺失，编译失败。

### 2. 解决方法

然后在`pom.xml`的plugins下添加如下配置，重点是`<compilerArguments>`参数。

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.6.0</version>
    <configuration>
        <source>1.8</source>
        <target>1.8</target>
        <compilerArguments>
            <extdirs>${project.basedir}\lib</extdirs>
        </compilerArguments>
    </configuration>
</plugin>
```





## 0x02 问题2：打包时lib中没有被添加

### 1. 问题描述

`compile`结束后进行打包，打包无异常，但是手动`java`命令执行指定的方法，此时提示library缺失。打开jar包发现lib目录下的jar包并没有被打包进去，导致实际运行时文件缺失，类加载器无法完成对类的加载。

### 2. 解决方法

然后一通QWER连招发现lib并没有被添加到maven的管理中，所以不会被打包。遂添加如下配置，将该jar包添加到pom文件中，使用scope为system中。

```xml
<dependency>
    <groupId>com.duoyi.hadoop</groupId>
    <artifactId>lzo</artifactId>
    <version>0.4.21</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/lib/hadoop-lzo-0.4.21-SNAPSHOT.jar</systemPath>
</dependency>
```

然并卵，`mvn assembly:single`打包也没有将该文件添加进去，说好的jar-with-dependencies呢。

最后决定使用assembly的自定义打包功能，在项目的根目录下添加`assembly.xml`文件。关于assembly的其他配置这里就不说了，google上有很多，要注意的配置有如下几个。

```xml
<!--assembly.xml-->
<dependencySets>
    <dependencySet>
        <outputDirectory>/</outputDirectory>
        <!--需要解包后才可以使用java执行-->
        <unpack>true</unpack>
    </dependencySet>
    <!--添加作用域为system中的jar文件，和pom文件中引入的本地依赖对应-->
    <dependencySet>
        <outputDirectory>/</outputDirectory>
        <unpack>true</unpack>
        <scope>system</scope>
    </dependencySet>
</dependencySets>
```

```xml
<archive>
    <manifest>
        <!--设置classpath-->
        <addClasspath>true</addClasspath>
        <classpathPrefix>lib/</classpathPrefix>
        <mainClass>
            com.duoyi.SparkMinuteCount
        </mainClass>
    </manifest>
    <!-- 添加本地的jar -->
    <manifestEntries>
        <!--多个包用空格隔开就可以了 -->
        <Class-Path>lib/hadoop-lzo-0.4.21-SNAPSHOT.jar</Class-Path>
    </manifestEntries>
</archive>
<!--使用的assembly.xml文件自定义打包-->
<descriptors>
    <descriptor>assembly.xml</descriptor>
</descriptors>
```

然后再执行编译打包，此时需要的依赖都已经被打包进去，如果有需要排除的或者额外添加的文件等等，可以在assembly.xml文件中进行添加配置即可。

此处再提一下，在archive中添加了上面写到的几个配置，但是不想写自定义的配置文件，想直接使用默认的`jar-with-dependencies`是不会起作用的。



上面描述的是将项目打包成可执行的jar包的方式，具体打包成web目录的war包没有研究，但是直接修改自定义的`assembly.xml`文件即可。



## 0x03 参考资料

[使用maven-assembly-plugin将第三方依赖jar包 打包入jar](https://my.oschina.net/hccake/blog/1552952)

[maven-assembly-plugin 入门指南](https://www.jianshu.com/p/14bcb17b99e0)

[maven assembly插件使用](http://ee-dreamer.com/?p=1129)

[maven-assembly-plugin 使用总结](https://zshell.cc/2016/11/19/tools-maven--assembly_plugin/)

[Apache Maven Assembly Plugin](http://maven.apache.org/plugins/maven-assembly-plugin/)





