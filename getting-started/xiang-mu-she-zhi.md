# 项目设置

## 依赖项

通过向现有项目添加`statefunk-sdk`或使用提供的maven原型，可以快速开始构建有状态函数应用程序。

```text
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>statefun-sdk</artifactId>
    <version>2.0.0</version>
</dependency>
```

## Maven原型

```text
$ mvn archetype:generate \
    -DarchetypeGroupId=org.apache.flink \
    -DarchetypeArtifactId=statefun-quickstart \
    -DarchetypeVersion=2.0.0
```

这样允许你为新创建的项目命名，交互式地询问groupId、artifactId和包名。之后会创建一个与artifact id同名的新目录。

```text
$ tree statefun-quickstart/
  statefun-quickstart/
  ├── Dockerfile
  ├── pom.xml
  └── src
      └── main
          ├── java
          │   └── org
          │       └── apache
          |            └── flink
          │             └── statefun
          │              └── Module.java
          └── resources
              └── META-INF
                └── services
                  └── org.apache.flink.statefun.sdk.spi.StatefulFunctionModule
```

该项目包含四个文件：

* `pom.xml`：具有基本依赖关系的pom文件，可开始构建有状态功能应用程序。
* `Module`：应用程序的入口点。
* `org.apache.flink.statefun.sdk.spi.StatefulFunctionModule`：用于运行时查找模块的服务实体。
* `Dockerfile`：一个Dockerfile，用于快速构建准备部署的有状态函数镜像。

我们建议将此项目导入到IDE中进行开发和测试。IntelliJ IDEA支持Maven项目开箱即用。如果使用Eclipse，则允许m2e插件导入Maven项目。某些Eclipse捆绑包默认包含该插件，而另一些则需要手动安装。

### 构建项目 <a id="build-project"></a>

如果要构建/打包项目，请转到项目目录并运行`mvn clean package`命令。命令会生成一个JAR文件，其中包含你的应用程序，以及可能添加为依赖项的所有库：`target/<artifact-id>-<version>.jar`。



