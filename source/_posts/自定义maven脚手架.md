---
title: 自定义maven脚手架
tags:
  - maven
categories:
  - maven
date: 2019-11-08 15:40:50
---
# 写在前面
开发新项目就需要搭建新工程，但是搭建新工程的这个过程是非常繁琐浪费时间的，并且不可避免的需要踩坑。更可怕的是，如果是在一个团队中，每新起一个项目都由不同的开发人员去自定义的搭建工程结构，那么对后续的统一管理，监控，运维简直是灾难。基于以上几点，团队内部其实是非常有必要搭建一个统一的脚手架来供统一使用

# 制作一个脚手架
下面我们就来详细的介绍如何搭建一个maven工程的脚手架

要搭建脚手架，首先我们需要一个模板工程，这个模板工程一般来说会集成一些工具类，底层中间件，通用配置，并且要有良好的分层结构等。需要能够达到开箱即用的程度。

以下是一个模板工程的目录结构，也是目前我们团队内部的标准化结构
```bash
.
├── example-client
│   ├── pom.xml
│   └── src
├── example-core
│   ├── pom.xml
│   └── src
├── example-server
│   ├── pom.xml
│   └── src
├── example-test
│   ├── pom.xml
│   └── src
├── Jenkinsfile
├── README.md
├── .gitignore
└── pom.xml
```

完成模板工程的搭建后，需要在模板工程的最外层 pom 文件加入以下配置

```xml
<build>  
    <pluginManagement>  
        <plugins>  
            <plugin>  
                <groupId>org.apache.maven.plugins</groupId>  
                <artifactId>maven-archetype-plugin</artifactId>  
                <version>3.0.1</version>  
            </plugin>  
            <plugin>  
                <groupId>org.apache.maven.plugins</groupId>  
                <artifactId>maven-compiler-plugin</artifactId>  
                <version>3.6.1</version>
                <configuration>  
                    <source>1.8</source>  
                    <target>1.8</target>  
                </configuration>  
            </plugin>  
            <plugin>  
                <groupId>org.apache.maven.plugins</groupId>  
                <artifactId>maven-resources-plugin</artifactId>  
                <version>3.0.2</version>
                <configuration>  
                    <encoding>UTF-8</encoding>  
                </configuration>  
            </plugin>  
        </plugins>  
    </pluginManagement>  
</build>
```

完成之后在根目录下执行

```bash
mvn archetype:create-from-project
```

刷新项目后，会发现在 `./target/generated-sources/archetype`目录下生成了脚手架工程，生成的脚手架工程可以当成是一个独立的项目，目录结构如下图

```bash
.
├── pom.xml
├── src
│   ├── main
│   │   └── resources
│   │       ├── META-INF
│   │       │   └── maven
│   │       │       └── archetype-metadata.xml
│   │       └── archetype-resources
│   │           ├── Jenkinsfile
│   │           ├── README.md
│   │           ├── __artifactId__.iml
│   │           ├── __rootArtifactId__-client
│   │           │   ├── __parentArtifactId__-client.iml
│   │           │   ├── pom.xml
│   │           │   └── src
│   │           ├── __rootArtifactId__-core
│   │           │   ├── __parentArtifactId__-core.iml
│   │           │   ├── pom.xml
│   │           │   └── src
│   │           ├── __rootArtifactId__-server
│   │           │   ├── __parentArtifactId__-server.iml
│   │           │   ├── pom.xml
│   │           │   └── src
│   │           ├── __rootArtifactId__-test
│   │           │   ├── __parentArtifactId__-test.iml
│   │           │   ├── pom.xml
│   │           │   └── src
│   │           └── pom.xml
│   └── test
│       └── resources
│           └── projects
│               └── basic
│                   ├── archetype.properties
│                   └── goal.txt

```

 在脚手架工程目录下执行 mvn install 就完成了脚手架的本地安装，安装完成之后，这个脚手架在本地就可以使用了
可以执行以下脚本来通过此脚手架创建项目

```bash
mvn archetype:generate -DgroupId=com.xxx.example -DartifactId=xxxx -Dpackage=com.xxx.example -DarchetypeGroupId=com.demo.archetype -DarchetypeArtifactId=demo-archetype -DarchetypeVersion=1.0.0-SNAPSHOT -DinteractiveMode=false
```

# 扫坑
不过如果我们直接这样使用的话，会发现生成了很多我们并不想要它出现的文件，比如 .idea .iml 文件等等，并且 .gitignore 文件也诡异的消失了（不知为何会忽略这个文件？？）... 这显然不是成熟的脚手架了。那么就需要对它做一些额外的配置了

有两种方式可以解决上面出现的问题

1. 将 .gitignore文件重命名为 __gitignore__，然后在模板工程根目录下新建 archetype.properties 文件，并填入以下内容

```bash
## generate for archetype-metadata.xml
excludePatterns=archetype.properties,*.iml,.idea/,.idea/libraries,logs/,build.sh
## generate .gitignore file
gitignore=.gitignore
```

完成上述配置后，重新执行 mvn archetype:create-from-project 生成脚手架工程。再完成本地安装，上面出现的问题就会解决

2. 第二种方式本质上和第一种方式是一样的，只是第二种方式是直接修改脚手架工程的配置文件。第一种方式相当于是执行  mvn archetype:create-from-project 时读取了 archetype.properties 帮我们做了配置文件的修改。

maven的脚手架工程下有两个重要的配置文件

- ./src/test/resources/projects/basic/archetype.properties
  - 这里可以加入自定义变量，如  gitignore 变量

- ./src/main/resources/META-INF/maven/archetype-metadata.xml

下面是一个典型的archetype-metadata.xml文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<archetype-descriptor xsi:schemaLocation="http://maven.apache.org/plugins/maven-archetype-plugin/archetype-descriptor/1.0.0 http://maven.apache.org/xsd/archetype-descriptor-1.0.0.xsd" name="example"
    xmlns="http://maven.apache.org/plugins/maven-archetype-plugin/archetype-descriptor/1.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <requiredProperties>
    <requiredProperty key="gitignore">
      <defaultValue>.gitignore</defaultValue>
    </requiredProperty>
    <requiredProperty key="port">
      <defaultValue>8888</defaultValue>
    </requiredProperty>
  </requiredProperties>
  <fileSets>
    <fileSet encoding="UTF-8" filtered="true">
      <directory></directory>
      <includes>
        <include>__gitignore__</include>
        <include>README.md</include>
        <include>Jenkinsfile</include>
        <include>pom.xml</include>
      </includes>
    </fileSet>
  </fileSets>
  <modules>
    <module id="${rootArtifactId}-client" dir="__rootArtifactId__-client" name="${rootArtifactId}-client">
      <fileSets>
        <fileSet filtered="true" packaged="true" encoding="UTF-8">
          <directory>src/main/java</directory>
          <includes>
            <include>**/*.java</include>
          </includes>
        </fileSet>
      </fileSets>
    </module>
    <module id="${rootArtifactId}-core" dir="__rootArtifactId__-core" name="${rootArtifactId}-core">
      <fileSets>
        <fileSet filtered="true" packaged="true" encoding="UTF-8">
          <directory>src/main/java</directory>
          <includes>
            <include>**/*.java</include>
          </includes>
        </fileSet>
        <fileSet filtered="true" encoding="UTF-8">
          <directory>src/main/resources</directory>
          <includes>
            <include>**/*.xml</include>
          </includes>
        </fileSet>
      </fileSets>
    </module>
    <module id="${rootArtifactId}-server" dir="__rootArtifactId__-server" name="${rootArtifactId}-server">
      <fileSets>
        <fileSet filtered="true" packaged="true" encoding="UTF-8">
          <directory>src/main/java</directory>
          <includes>
            <include>**/*.java</include>
          </includes>
        </fileSet>
        <fileSet filtered="true" encoding="UTF-8">
          <directory>src/main/resources</directory>
          <includes>
            <include>**/*.xml</include>
          </includes>
        </fileSet>
        <fileSet filtered="true" encoding="UTF-8">
          <directory>src/main/resources/META-INF/dubbo</directory>
          <includes>
            <include>*</include>
          </includes>
        </fileSet>
        <fileSet filtered="true" encoding="UTF-8">
          <directory>src/main/resources</directory>
          <includes>
            <include>**/*.yml</include>
          </includes>
        </fileSet>
      </fileSets>
    </module>
    <module id="${rootArtifactId}-test" dir="__rootArtifactId__-test" name="${rootArtifactId}-test">
      <fileSets>
      </fileSets>
    </module>
  </modules>
</archetype-descriptor>

```

## archetype-metadata.xml 结构
简单介绍archetype-metadata.xml文件的基本结构

- `<requiredProperties>`是属性变量定义层，在这里定义的变量可以在archetype工程文件中通过 `${xxx}` 来进行引用，文件名则可以通过 `__xxx__`来引用变量（这就是为什么要把`.gitignore`重命名为`__gitignore__`）。变量在执行 `mvn archetype:generate`时才录入真正内容
- 再往下其实就是定义将要生成的工程目录结构，首先是一个 `fileSets` （包含多个 `fileSet`标签）标签定义了根目录需要生成哪些文件，再通过多个` modules` 标签来定义多个子模块（如果是多模块工程的话）。这其中比较重要的其实还是 `fileSet`标签，fileSet有两个重要的属性
    - filtered = true 如果为true，则该 `fileSet`包含的文件中的` ${}` 占位符会被替换成相应的变量
    - packaged = true 如果为true，则 `src/main/java` 下得文件内容会被加入 指定包路径下



# 总结

一个优秀少坑的脚手架还是能极大的提升生产力的，毕竟这种重复且无价值的劳动我们还是交给工具吧

