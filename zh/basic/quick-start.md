## 建构应用
创建应用是一个最普通的任务，每个应用都需要编写一个入口函数，该函数通常需要解析输入参数并初始化应用提供的各种服务。

UAPI已经提供一个名为Bootstrap的类来处理应用启动应用，它会初始化基本的服务，解析输入参数，并且发送SystemStartingUpEvent事件来通知应用启动。

下面我们从创建一个最基本的应用来了解UAPI的基本使用，该应用输出Hello Word到控制台上。

### 设置依赖
没有应用是独立存在的，通常它需要依赖一些外部的代码库，所以开发者需要在建构文件中定义依赖。
Java世界中有很多用来管理以来的的建构系统，这里我们使用Gradle方式来定义我们第一个应用的依赖关系。
```gradle
repositories {
    mavenCentral()
    jcenter()
    maven { url "http://dl.bintray.com/typesafe/maven-releases" }
    maven { url "http://dl.bintray.com/inactionware/maven-release" }
    maven { url "http://dl.bintray.com/inactionware/maven-snapshot" }
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

### 创建服务
服务是UAPI框架的最基本的单元，服务其实就是一个带有一些特殊注解的java类，我们第一个应用就使用服务输出一个“Hello World”到控制台上。

```java
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
这是个非常简单的java类，它在activate方法中打印了“Hello World”到日志（日志默认是输出到控制台）。该类还包含了一些注解，通过这些注解UAPI框架会对相应的java类做增强，其中：
* Service注解表明类该类将由UAPI进行管理，autoActivate属性指明了一旦该类所有的依赖满足，那么UAPI框架需要立即激活该类，对应的就是会调用下方activate方法，因为该方法被标注为OnActivate。
* Tag注解是该服务的绑定的标签，而标签可以用于Profile，通过Profile你可以指定那些服务可以被UAPI管理或者那些服务需要被忽略。
* Inject注解是告知UAPI框架该服务依赖另一个服务，并通过注解的字段进行注入。

### 创建配置文件
每个应用或多或少总会有配置文件，我们的第一个应用也不例外，UAPI支持多种配置文件，比如json，yaml配置文件，这里我们使用yaml格式的配置文件：
```yaml
app:
  active-profile: HelloAppProfile

profiles:
  - name: HelloAppProfile
    model: include
    matching: satisfy-any
    tags:
      - Hello
```
该配置文件比较简单，主要分两部分，最上面部分指出了应用当前激活的profile，而下半部分则定义了profile，包括了profile的名称，匹配机制以及profile对应的服务标签。

### 启动应用

## UAPI版本的你好，世界！
