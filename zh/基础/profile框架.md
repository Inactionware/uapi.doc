# 概述

在通常情况下，每个应用都会包含成千上万个服务，这些服务支持了应用对外提供的功能。在默认情况下，框架会注册所有的服务，并且在必要时激活这些服务，但是在某些情况下，我们或许只需要注册某些服务，或者禁止某些服务，这时候我们需要使用服务profile的特性。

# 服务标签

每个服务都可以定义该服务所关联的标签。

```java
@Service(autoActive = true)
@Tag("MyTag1")
public class TaggedService1 {

    @Inject
    protected ILogger _logger;

    @OnActivate
    public void onActivate() {
        this._logger.info("Tagged Service1 activated.");
    }
}

@Service(autoActive = true)
@Tag("MyTag2")
public class TaggedService2 {

    @Inject
    protected ILogger _logger;

    @OnActivate
    public void onActivate() {
        this._logger.info("Tagged Service2 activated.");
    }
}
```

通过`@Tag`来指定服务的标签，`@Tag`可以指定一个或者多个标签。
上面的例子我们定义了两个服务，分别对应`MyTag1`和`MyTag2`。
当然仅仅定义服务标签是没有任何功能的，服务标签需要与profile组合起来才有作用。

# 定义profile
Profile是一组配置，通过profile可以定义哪些tag应该被归类为同一个profile之中。
一个应用可以定义多个profile。
```yaml
profiles:
  - name: MyProfile1
    model: include
    matching: satisfy-any
    tags:
      - MyTag1
  - name: MyProfile2
    model: include
    matching: satisfy-any
    tags:
      - MyTag2
```
这个配置文件里面定义了两个profile，分别是`MyProfile1`和`MyProfile2`。
Profile有多个属性：
* name属性定义了该profile的名称，不能和其他profile的名称相同。
* model属性定义了服务标识的匹配模式，也就是匹配的服务是包含还是排除在注册服务中，可选的值有`include`和`exclude`。
* matching属性定义了服务标识是如何匹配的，可选的值有`satisfy-any`和`satisfy-all`，`satisfy-any`指明了profile中定义的标签只要有一个和服务定义的标签匹配，那么该服务就匹配了该profile。`satify-all`指明了profile中的所有标签必须和服务定义标签匹配，则改服务就匹配了该profile。
* tags属性定义了一个或多个标签的列表。

# 指定激活的profile
每个应用能够定义一个激活的profile：
```yaml
app:
  active-profile: MyProfile1
```

启动应用，你会看到如下输出：
```shell
14:23:13.840 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config update system.config -> conf/activate-profile.yaml
14:23:13.902 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/activate-profile.yaml
14:23:13.934 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - Application is going to startup...
14:23:13.934 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - MyProfile1
14:23:13.934 [ForkJoinPool.commonPool-worker-1] INFO  uapi.tutorial.profile.TaggedService1 - Tagged Service1 activated.
14:23:13.934 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - The application is launched
14:23:13.934 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.Application - Application startup success.
14:23:13.934 [ForkJoinPool-1-worker-1] WARN  uapi.event.internal.EventBus - There are no event handler for event topic - ApplicationStartup
```
从日志中可以看出`MyProfile1`在应用启动的时候被激活了，所以`TaggedService1`启动的时候被激活了，并打印出了`Tagged Service1 activated.`的日志。
如果将`activate-profile`设置为`MyProfile1`的话，那么`TaggedService2`将在启动的时候被激活。