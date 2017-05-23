# 创建应用
每个Java应用都需要一个main函数作为应用的入口，然后需要解析启动参数，初始化服务等等。
UAPI框架已经提供一个名为Bootstrap的类来处理这些通用任务，它会抛出AppStartupEvent事件来通知应用已经启动完成，你可以通过注册该事件的处理函数来添加应用启动后的业务逻辑。

```java
@Service(autoActive = true)
@Tag("HumanApp")
public class Human {

    @Inject
    protected IResponsibleRegistry _respReg;

    @OnActivate
    public void activate() {
        this._respReg.register("Mike")
                .newBehavior("Say Hello", AppStartupEvent.class, AppStartupEvent.TOPIC)
                .then(Greeting.actionId)
                .build();
    }

    @Service
    @Action
    @Tag("HumanApp")
    public static class Greeting {

        public static final ActionIdentify actionId = ActionIdentify.toActionId(Greeting.class);

        @Inject
        protected ILogger _logger;

        @ActionDo
        public void say(AppStartupEvent event, IExecutionContext execCtx) {
            this._logger.info("Hello from {}!", execCtx.responsibleName());
        }
    }
}
```
这里我们创建了一个Human类，当应用启动时它会被自动激活（autoActive=true），UAPI框架会调用OnActivate注解的方法，该方法要求是一个没有参数返回void的方法。
你可以定义多个OnActivate方法，UAPI框架会逐一调用，但是UAPI框架并不保证其调用的顺序，因此你的逻辑不能对调用顺序有任何依赖。
在activate方法中，我们注册了一个名为Mike的Responsible，并给Mike创建了一个新的behavior，这个behavior的输入是AppStartupEvent事件，输出为void（更多关于该行为框架的细节可参见后续章节）。
Human类还包含了一个Greeting的内部类（注意，如果内部声明为一个Service，那么它必须是静态的），该内部类是个Action（由Action注解），因此它需要提供一个被ActionDo注解的方法，该方法就是该Action具体执行的逻辑。
ActionDo注释的方法的参数可以是一个或者两个，如果是一个的话，那么这个参数就是该Action的输入参数，如果是两个的话，第二个参数必须是IExecutionContext类型，IExecutionContext接口提供了一些执行该Action时的上下文的信息。

# 结束应用
一个应用有启动，当然也应该就终止，在UAPI中有两种方式结束应用：
* 通过Ctrl+C来给应用发送终止信号，UAPI框架接收到该信号会调用停止应用的处理函数，这时候每个服务的中如果有方法是标注OnDeactivate注解的则会被调用。
* 通过创建ExitSystemRequest事件来请求UAPI框架停止应用，和上一种方法一样UAPI框架也会进行调用应用停止的处理函数。

下面的代码片段展示了通过发送ExitSystemRequest事件来终止应用：
```java
/**
 * A Terminator can send an event to terminate application
 */
@Service(autoActive = true)
@Tag("Terminator")
public class Terminator {

    @Inject
    protected IResponsibleRegistry _respReg;

    @Inject
    protected ILogger _logger;

    @OnActivate
    public void activate() {
        IResponsible resp = this._respReg.register("Terminator");
        resp.newBehavior("Do Something", AppStartupEvent.class, AppStartupEvent.TOPIC)
                .then(StartUp.actionId)
                .onSuccess((input, ctx) -> new ExitSystemRequest("Terminator"))
                .build();
        resp.newBehavior("Shutdown flow", AppShutdownEvent.class, AppShutdownEvent.TOPIC)
                .then(CleanUp.actionId)
                .build();
    }

    @OnDeactivate
    public void deactivate() {
        this._logger.info("The service is deactivated - {}", Terminator.class.getName());
    }

    @Service
    @Action
    @Tag("Terminator")
    public static class StartUp {

        private static final ActionIdentify actionId = ActionIdentify.toActionId(StartUp.class);

        @Inject
        protected ILogger _logger;

        @ActionDo
        public void doSomething(AppStartupEvent event) throws Exception {
            this._logger.info("Do start up action");
            Thread.sleep(1000);
        }
    }

    @Service
    @Action
    @Tag("Terminator")
    public static class CleanUp {

        private static final ActionIdentify actionId = ActionIdentify.toActionId(CleanUp.class);

        @Inject
        protected ILogger _logger;

        @ActionDo
        public void cleanUp(AppShutdownEvent event) {
            this._logger.info("Do clean up action");
        }
    }
}
```
该代码当应用启动后，会调用StartUp的Action，该Action会输出一行日志，然后让线程睡眠一秒钟，然后当该Behavior处理结束后会发送ExitSystemRequest的请求，当UAPI接收到该请求后启动退出系统的动作。
退出动作首先会发送AppShutdownEvent，所有需要清理资源的Responsible都需要定义相应的Behavior来处理资源清理的过程，当所有的资源清理的Behavior都执行完毕后，UAPI框架会调用所有Service的OnDeactivate注解标注的方法。

上述代码执行的输出：
```
10:36:55.875 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config update cli.config -> conf/terminate-app-config.yaml
10:36:55.937 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/terminate-app-config.yaml
10:36:55.953 [ForkJoinPool-1-worker-1] INFO  u.a.i.ApplicationConstructor$StartupApplication - Application is going to startup...
10:36:55.953 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - TerminateAppProfile
10:36:55.969 [ForkJoinPool-1-worker-1] INFO  u.a.i.ApplicationConstructor$StartupApplication - The application is launched
10:36:55.969 [ForkJoinPool-1-worker-1] INFO  u.a.internal.ApplicationConstructor - Application startup success.
10:36:55.969 [ForkJoinPool-1-worker-1] INFO  uapi.tutorial.app.Terminator$StartUp - Do start up action
10:36:56.975 [ForkJoinPool-1-worker-1] INFO  u.a.internal.ApplicationConstructor - Application is going to shutdown...
10:36:56.975 [ForkJoinPool-1-worker-2] INFO  uapi.tutorial.app.Terminator$CleanUp - Do clean up action

Process finished with exit code 0
```

# 总结
当应用启动时，UAPI框架提供了AppStartupEvent，当应用终止时，UAPI框架提供了AppShutdownEvent。
通过发出ExitSystemRequest事件来主动请求UAPI框架终止应用。
