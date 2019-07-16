# Maven通用工程模板方案

> 本文旨在介绍使用`maven archetype`进行项目模板管理的方案，同时会介绍一些常用于质量检测的`maven`插件。

工作中常常会遇到这么一种情况：一个产品线或工作组内，会创建数十个甚至上百个结构高度相似，各自承担不同业务的应用工程项目。

比如一个常见的基于`spring-boot`的应用工程模块结构可能如下

```
-root
  |- doc         项目文档与阅读资料
  |- facade      接口与模型发布包
  |- app         实际服务进程代码
  |- checkstyle  项目风格检测工具
  |- pom.xml     项目依赖配置与版本管理配置
```

以如上工程结构为例，`pom.xml`和`checkstyle`通常是高度一致的; 

有时我们需要在研发团队要统一一个相对标准的项目构建结构(以及流水线方案), 比起让各个项目负责人在项目伊始复制粘贴的方案，
使用`archetype jar`进行模板项目的管理和创建则更为友好。

![workflow](doc/archetype-workflow.png)

## 1. `archetype`概述

简单地说，`archetype`是一个maven的模板工具，`archetype`定义了一组统配模型，用于构建大量同质的项目结构。
`Archetype`可以帮助我们去创建一个供其他研发人员使用的`maven`项目模板，甚至允许我们和依赖包一样，在仓库中对模板进行版本管理和发布。

使用`maven-archetype-plugin`，我们能够将一个已有的项目转化为`archetype project`。

![archetype](doc/archetype-overview.png)


通过生成的`archetype project`进行构建，就能生成`archetype jar`制品; 

由于`archetype project`也是一个`maven`项目，因此和其他的`dependency jar`制品一样，
我们也可以通过`install`或`deploy`等指令在本地安装或者发布至远端。

## 2. 转化模板工程

我们以一个简单的`springboot`模板项目为例, 可以在github上找到该[项目地址](https://github.com/my-helloworld/template-springboot)

该模板结构如下

```
-root
  |-app         SpringBoot应用
  |-checkstyle  项目风格检测工具
  |-doc         项目文档与阅读资料
  |-pom.xml     项目依赖配置与版本管理配置
```

项目结构本身很简单，唯一定制的部分是构建过程要求做一些常见的静态检测，如`checkstyle`与`findbugs`

接下来我们需要将这个项目转化为一个`archetype project`;

> 指定`archetype`构建配置

在项目的根目录下`pom.xml`声明配置:

```xml
<build>
    <extensions>
        <extension>
            <groupId>org.apache.maven.archetype</groupId>
            <artifactId>archetype-packaging</artifactId>
            <version>${ext.archetype.packaging.version}</version>
        </extension>
    </extensions>
</build>
```

> 构建`archetype project`

在项目根目录下执行生成archetype

```
mvn archetype:create-from-project
```

该指令会在`${project.build.directory}/generated-sources/archetype`下生成模板工程`archetype project`

```
target/generated-sources/archetype
|-- pom.xml
`-- src
    |-- main
    |   `-- resources
    |       |-- archetype-resources
    |       |   |-- pom.xml
    |       |   `-- src
    |       |       |-- main
    |       |       |   `-- java
    |       |       |       `-- App.java
    |       |       `-- test
    |       |           `-- java
    |       |               `-- AppTest.java
    |       `-- META-INF
    |           `-- maven
    |               `-- archetype-metadata.xml
    `-- test
        `-- resources
            `-- projects
                `-- it-basic
                    |-- archetype.properties
                    `-- goal.txt
```

## 3 本地安装与使用

在第一步中，我们生成了`artifact project`，为了使用这个模板，我们需要在本地进行安装

> 安装本地仓库

和所有的源码发布流程一样，archetype生成了一个支持发布的pom配置, 因此只需要执行`install`即可

```
cd target/generated-sources/archetype
mvn install
```

安装完毕后，能在本地的`~/.m2/repository/archetype-catalog.xml`中找到我们安装的`archetype`信息

```xml
<?xml version="1.0" encoding="UTF-8"?>
<archetype-catalog xsi:schemaLocation="http://maven.apache.org/plugins/maven-archetype-plugin/archetype-catalog/1.0.0 
    http://maven.apache.org/xsd/archetype-catalog-1.0.0.xsd"
    xmlns="http://maven.apache.org/plugins/maven-archetype-plugin/archetype-catalog/1.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <archetypes>
    <archetype>
      <groupId>com.chpengzh</groupId>
      <artifactId>template-springboot-archetype</artifactId>
      <version>latest</version>
      <description>template-springboot-archetype</description>
    </archetype>
  </archetypes>
</archetype-catalog>

```

> 使用本地模板创建项目

使用交互式命令行创建就能快速创建一个新的项目

```
mvn archetype:generate -DarchetypeCatalog=local
...
Choose archetype:
1: local -> com.chpengzh:template-springboot-archetype (template-springboot-archetype)
Choose a number or apply filter (format: [groupId:]artifactId, case sensitive contains): : 1  
Define value for property 'groupId': com.alipay.sofa
Define value for property 'artifactId': a-new-project      
Define value for property 'version' 1.0-SNAPSHOT: : 
Define value for property 'package' com.alipay.sofa: : 
Confirm properties configuration:
groupId: com.alipay.sofa
artifactId: a-new-project
version: 1.0-SNAPSHOT
package: com.alipay.sofa
 Y: : y
...
```

在shell场景，我们也可以使用预设环境变量的方式避免交互

```
yes | mvn archetype:generate \
    -DarchetypeCatalog=local \
    -DarchetypeGroupId=com.chpengzh \
    -DarchetypeArtifactId=template-springboot-archetype \
    -DarchetypeVersion=latest \
    -DgroupId=com.alipay.sofa \
    -DartifactId=a-new-project \
    -Dversion=1.0
```

## 4. 发布与使用

和安装过程一致，要发布一个`archetype jar`的过程只需要执行`mvn deploy`即可；

关于发布的具体流程，请参阅[Deploy Plugin](https://maven.apache.org/plugins/maven-deploy-plugin/)，本章节不详细讲解;

> 发布archetype

修改`target/generated-sources/archetype/pom.xml`

```xml
<distributionManagement>
    <repository>
        <id>archetype-release</id>
        <name>Archetype Releases</name>
        <url>http://xxx</url>
    </repository>
    <snapshotRepository>
        <id>archetype-snapshot</id>
        <name>Archetype Snapshots</name>
        <url>http://xxx</url>
    </snapshotRepository>
</distributionManagement>
```

在`~/.m2/settings.xml`中添加授权信息(略)

发布

```
cd target/generated-sources/archetype
mvn deploy
```

> 使用发布的模板

交互过程同本地创建

```
mvn archetype:generate \
    -DarchetypeCatalog=http://username:password@xxx/nexus/content/repositories/myapps/archetype-catalog.xml
```

## 阅读材料

- [Apache Maven Project: maven-deploy-plugin](https://maven.apache.org/plugins/maven-deploy-plugin/)
- [Apache Maven Project: introduction-to-archetypes](https://maven.apache.org/guides/introduction/introduction-to-archetypes.html)
- [创建自定义的archetype](https://xtuhcy.iteye.com/blog/1815061)