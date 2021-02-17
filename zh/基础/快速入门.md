# 建构应用
创建应用是一个最普通的任务，每个应用都需要编写一个入口函数，该函数通常需要解析输入参数并初始化应用提供的各种服务。

UAPI已经提供一个名为Bootstrap的类来处理应用启动应用，它会初始化基本的服务，解析输入参数，并且发送SystemStartingUpEvent事件来通知应用启动。

下面我们从创建一个最基本的应用来了解UAPI的基本使用，该应用输出Hello Word到控制台上。

## 设置依赖
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

## 创建服务
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
* OnActivate是和服务生命周期关联的注解，表示当服务激活的时候调用该方法，类似的注解还有OnDeactivate。

> 注意，和Spring的运行时增强不同，UAPI使用的是编译时对类进行增强，因此任何使用或修改UAPI框架提供的注解，必须对该类文件进行重新编译，否则会在运行时抛出异常

## 创建配置文件
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

## 启动应用
UAPI的框架的入口类是uapi.app.Bootstrap，因此你可以通过如下方式启动应用：
```shell
java -cp [class path] uapi.app.Bootstrap -config=[config file path]
```
一旦应用成功启动，你将会看到控制台输出如下日志内容：
```
11:09:36.789 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config update cli.config -> conf/hello-config.yaml
11:09:36.851 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/hello-config.yaml
11:09:36.882 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - HelloAppProfile
11:09:36.882 [ForkJoinPool.commonPool-worker-1] INFO  uapi.tutorial.quickstart.HelloWorld - Hello World
11:09:36.882 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.StartupApplication - The application is launched
11:09:36.882 [ForkJoinPool-1-worker-1] WARN  uapi.event.internal.EventBus - There are no event handler for event topic - ApplicationStartup
```
通过该日志内容你可以了解一些基本情况：
* 日志第一行表明UAPI框架从命令行参数中获取了一个配置，所有命令行参数转换到UAPI框架中都会添加"cli."的前缀。
* 日志第三行表明了当前激活的profile是HelloAppProfile。
* 日志第四行表明了UAPI框架激活了HelloWorld服务，并调用其activate方法，该方法在日志里面打印了"Helo World"字符串。
* 日志第五行指出了该应用已经成功启动
* 日志第六行是一个警告信息，表示一个名为ApplicationStartup的事件没有对应的事件处理器来处理。

> 注意：profile不是必须要提供的，如果没有指定profile，则UAPI则会激活一个默认的profile，该默认profile会加载所有的service。

# UAPI版本的HelloWorld
上面例子是通过Service的生命周期回调方法来输出Hello World到日志的，当然这是允许的，但这并不是UAPI框架推荐的方式。

在开发应用的时候，我们通常会对应用进行领域建模，会得到领域对象，进而规划出这些领域对象会有哪些功能，所以UAPI也使用这种方式。
在UAPI框架里领域对象被称为Responsible，每个Responsible可以绑定多个Behavior（行为），而Action（动作）是组成Behavior的最小单位，Behavior一般由Event（事件）来触发的。

那么我们开始对我们的Hello World进行领域建模，输出Hello World是个简单的动作，但是动作总会有个动作主体，那就是领域对象，我们这个需要诞生一个小孩子，当小孩子出生的时候会对这个新世界打一声招呼（Hello World）。

我们先创建一个活动，用于输出Hello World：
```java
/**
 * A Greeting will output hello word to the logger
 */
@Service
@Action
@Tag("BabyHello")
public class Greeting {

    public static final ActionIdentify actionId = ActionIdentify.parse(
            StringHelper.makeString("{}@{}", Greeting.class.getName(), ActionType.ACTION));

    @Inject
    protected ILogger _logger;

    @ActionDo
    public void say(AppStartupEvent event, IExecutionContext execCtx) {
        this._logger.info("Hello World form {}", execCtx.responsibleName());
    }
}
```
这个类和前面的HelloWorld例子中的类非常像，但多了些不同的东西：
* Action注解表明了该服务是一个动作，对应的ActionDo注解指出该动作执行的方法
* 每个动作有个标识，默认是service类的类名

下面我们来创建Responsible:
```java
/**
 * Hello world application demo
 */
@Service(autoActive=true)
@Tag("BabyHello")
public class BabyCreator {

    @Inject
    protected IResponsibleRegistry _responsibleReg;

    @OnActivate
    public void activate() {
        IResponsible baby = this._responsibleReg.register("Baby");
        baby.newBehavior("Say Hello", AppStartupEvent.class, AppStartupEvent.TOPIC)
                .then(Greeting.actionId)
                .traceable(true)
                .build();
    }
}
```
BabyCreator服务注入了一个IResponsibleRegistry的外部服务，该服务用来创建和管理Responsible的。
在activate方法中，通过IResponsibleRegistry注册了一个名为Baby的Responsible，然后对该Responsible创建了一个吗名为Say Hello的新Behavior，该Behavior的输入是AppStartupEvent事件。
Say Hello的Behavior只包含了一个Action，就是Greeting。

启动该应用，你会看到如下输出：
```
13:57:39.817 [ForkJoinPool.commonPool-worker-2] INFO  u.c.internal.FileBasedConfigProvider - Config update cli.config -> conf/baby-hello-config.yaml
13:57:39.864 [ForkJoinPool.commonPool-worker-2] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/baby-hello-config.yaml
13:57:39.895 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - BabyHelloAppProfile
13:57:39.895 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.StartupApplication - The application is launched
13:57:39.895 [ForkJoinPool-1-worker-1] INFO  uapi.tutorial.quickstart.Greeting - Hello World form Baby
```
本例的输出和上一个例子基本类似，只是Hello World的输出已经是在应用启动后才输出的。

UAPI版本的Hello World看上去要比传统的Hello World要复杂的多，为什么还需要这样做呢？其实本身Hello World如此简单的例子无需UAPI框架，这里只是为了展示一些UAPI框架的基础感念。
使用UAPI框架的事件行为框架我们可以更容易将业务逻辑代码从业务数据中分离出来，你可以看到Greeting这个Action是高度可重用的，只要符合指定接口，任意Responsible都可以使用它。
其次通过Behavior我们可以插入任意多的Action来对已有的Behavior进行增强，只要它们符合Action的接口。

# 总结
本篇文章只要概要介绍了几个UAPI框架的重要概念，应用，服务，事件，行为等，这只是UAPI框架的冰山一角，后续的教程将会详细讨论这些概念的每个细节。

本章的样例代码可以在[uapi.tutorial.quickstart](https://github.com/Inactionware/uapi.tutorial/tree/master/uapi.tutorial.quickstart)中找到。
