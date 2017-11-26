# 概述

UAPI(Universal Application Platform Infrastructure)框架旨在为开发者提供了一组用于搭建应用程序的基础框架，利用该框架开发者可以更容易关注业务逻辑的实现而无需关心一些底层通用逻辑的实现细节。

何谓框架？与程序库有何区别？
框架指的是对应用程序中某一个通用领域的业务流程抽象，而程序库则是一些通用的业务逻辑的组合，两者有本质的区别。大多数框架都是由程序库进化而来的。

以面向对象设计来说，当需要开发一个应用的时候，我们会对我们的业务进行领域建模，提取出领域对象，然后设计出领域对象可接受的消息及其状态是如何受消息影响的，当然我们还会规划出各个领域对象是如何通过发送消息来进行交互的模型，这个模型最终实现了我们应用的各种需求。

在众多领域对象中我们会发现有一些领域对象包含了一些非常通用的逻辑，它们被大多数其他的领域对象所依赖，这些领域对象就变成了程序库，例如Java里面的Collections就是典型的程序库，它提供了集合的功能。

# 概念
UAPI框架提供了服务依赖管理，例如：
```Java
/**
 * Hello world application demo
 */
@Service(autoActive=true)
@Tag("Hello")
public class HelloWorld {

    @Inject
    protected ILogger _logger;

    @OnActivate
    public void activate() {
        this._logger.info("Hello World");
    }
}
```
HelloWorld服务依赖ILogger服务，UAPI框架在运行时可以为HelloWorld服务注入ILogger服务。

UAPI框架还提供了易于注入的配置服务，例如：
TODO

利用UAPI框架提供的事件行为框架，使得应用程序能够容易地划分成多个逻辑处理块，每个逻辑块可以重复使用, 例如：
TODO

# 使用
为了引入UAPI相关的包，你必须添加UAPI的仓库，在gradle中：
```groovy
repositories {
    maven { url "http://dl.bintray.com/inactionware/maven-snapshot" }
    maven { url "http://dl.bintray.com/inactionware/maven-release" }
}
```

通过maven或者gradle来增加对UAPI框架的依赖，使用gradle：
```groovy
dependencies {
    compile "uapi:uapi.service:${uapi_cornerstone_version}"
    compile "uapi:uapi.config:${uapi_cornerstone_version}"
}
```
UAPI框架由多个java包组成，你可以指定你需要的包来导入。

# 编译UAPI
使用git clone UAPI项目到本地，然后使用下面命令来编译：
```shell
./gradlew clean build
```
该命令会自动下载gradle，然后使用gradle来编译UAPI项目。

> 注意，编译需要JDK1.8+，因此你需要下载安装JDK1.8+，在项目的根目录有个名为settings.env（windows使用settings.env.bat），在该文件中设置JAVA_HOME环境变量，然后使用`source ./settings.env`执行该文件来设置环境变量（windows直接在命令行中执行settings.env.bat即可）。
