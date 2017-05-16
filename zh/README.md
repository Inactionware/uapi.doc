# 概述

UAPI(Universal Application Platform Infrastructure)框架旨在为开发者提供了一组用于搭建应用程序的基础库，利用该框架开发者可以更容易关注业务逻辑的实现而无需关心一些底层通用逻辑的实现细节。

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

> 注意，编译需要JDK1.8+，因此你需要下载安装JDK1.8+，在项目的根目录有个名为settings.env（windows使用settings.env.bat），在该文件中设置JAVA_HOME环境变量，然后运行该文件。
