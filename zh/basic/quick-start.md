## 建构应用
创建应用是一个最普通的任务，每个应用都需要编写一个入口函数，该函数通常需要解析输入参数并初始化应用提供的各种服务。

UAPI已经提供一个名为Bootstrap的类来处理应用启动应用，它会初始化基本的服务，解析输入参数，并且发送SystemStartingUpEvent事件来通知应用启动。

下面我们从启动一个最基本的应用来输出Hello Word到控制台上。

### 设置建构依赖
没有应用是独立存在的，通常它需要依赖一些外部的代码库，所以开发者需要在建构文件中定义依赖。
Java世界中有很多用来管理以来的的建构系统，这里我们使用Gradle方式来定义我们第一个应用的依赖关系。
```gradle
repositories {
    mavenCentral()
    jcenter()
    maven { url "http://dl.bintray.com/typesafe/maven-releases" }
    maven { url "http://dl.bintray.com/inactionware/maven-release" }
}

dependencies {
    compile "uapi:uapi.service:${uapi_cornerstone_version}"
    compile "uapi:uapi.config:${uapi_cornerstone_version}"
    compile "uapi:uapi.log:${uapi_cornerstone_version}"
    compile "uapi:uapi.event:${uapi_cornerstone_version}"
    compile "uapi:uapi.behavior:${uapi_cornerstone_version}"
    compile "uapi:uapi.app:${uapi_cornerstone_version}"
}
```
在build.gradle文件中，我们首先定义了几个仓库，其中inactionware的仓库是用来获取UAPI框架相关的jar文件。
在仓库定义的下面，我们定义了我们应用程序的依赖项目，基本都是对UAPI框架的依赖。其中uapi_cornerstone_version是一个变量，保存的依赖版本信息，这些变量定义在gradle.properties文件中：
```properties
uapi_cornerstone_version    = 1.0.0+
```

## 你好，世界！

## UAPI版本的你好，世界！
