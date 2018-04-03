# 概念

在设计软件架构时，我们会对需要实现的功能进行领域建模，抽象出领域模型以及模型上的功能行为，比如在一个权限管理系统里面，我们会定义几个领域模型：用户，角色，权限等，然后赋予用户可以关联一个或者多个角色，而角色可以赋予多个权限，从而使得用户在访问某个资源时系统可以针对该用户进行权限认证。

领域模型和赋予模型之上的行为是软件设计中必不可少的过程，因此我们抽象出这些内容从而形成了行为框架，通过行为框架是的领域建模更简单易行。

几个基本的概念：

* 责任者：框架中称之为`Responsible`，它相当于领域模型本身。
* 活动：框架中称之为`Action`组成行为的最基本的单位，它包含了一段逻辑用于处理输入数据并产生输出数据。
* 行为：框架中称之为`Behavior`，由一个或多个活动组成，可以被关联到某个责任者上。

# 定义活动

通过`@Action`来定义一个活动：

```java
@Service
@Action
@Tag("SimpleBehavior")
public class SimpleAction {

    public static final ActionIdentify actionId = ActionIdentify.toActionId(SimpleAction.class);

    @Inject
    protected ILogger _logger;

    @ActionDo
    public void execute(AppStartupEvent event, IExecutionContext execCtx) {
        this._logger.info("Execute in SimpleAction by {}.", execCtx.behaviorName());
    }
}
```

上面代码定义了一个活动，用`@ActionDo`来标识某个方法是活动执行的代码逻辑，这个方法可以接受1个或2个参数，第一个参数是该活动的输入数据类型，第二个参数必须是`IExecutionContext`类型，用它可以获取活动执行上下文的信息，例如是哪个责任者执行的该活动。

# 定义行为

定义行为比较简单：

```java
@Service(autoActive = true)
@Tag("SimpleBehavior")
public class SimpleApp {

    @Inject
    protected IResponsibleRegistry _respReg;

    @OnActivate
    public void activate() {
        this._respReg.register("A Responsible")
                .newBehavior("Simple Behavior", AppStartupEvent.class, AppStartupEvent.TOPIC)
                .then(SimpleAction.actionId)
                .build();
    }
}
```

因为行为通常是需要绑定责任者的，所以上面代码首先声明需要注入`IResponsibleRegistry`服务，这个服务用于注册责任者，然后在`activate`方法中注册了一个名为`A Responsible`的责任者，该责任者定义了一个名为`Simple Behavior`的行为，该行为的输入数据是`AppStartupEvent`类型的数据（它其实就是应用启动的事件）。接下来用`then`方法为该行为绑定第一个活动（也是最后一个活动）`SimpleAction`，然后创建该行为。这里需要注意行为的第一个活动的输入数据类型必须与行为声明的输入数据的类型一致，否则运行时会抛出异常。

运行该程序，你可以得到以下输出：

```shell
15:05:34.306 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config update system.config -> conf/simple-behavior.yaml
15:05:34.390 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/simple-behavior.yaml
15:05:34.428 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - Application is going to startup...
15:05:34.429 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - SimpleBehaviorProfile
15:05:38.086 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - The application is launched
15:05:38.086 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.Application - Application startup success.
15:05:38.087 [ForkJoinPool-1-worker-2] INFO  uapi.tutorial.behavior.SimpleAction - Execute in SimpleAction by Simple Behavior.
```

可以看到当系统启动后，`SimpleAction`的到了执行，并打印出`Execute in SimpleAction by Simple Behavior.`的日志信息。

# 事件和行为的绑定