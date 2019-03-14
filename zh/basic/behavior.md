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

上面代码定义了一个活动，用`@ActionDo`来标识某个方法是活动执行的代码逻辑，这个方法可以接受零个或多个参数，参数有三种类型：
1. 上下文参数，该参数必须为```IExecutionContext```类型，通过该参数获取活动执行时的上下文信息，例如该活动所属的行为，责任者等信息。
1. 活动的输出参数，该参数可以有多个，其类型必须为```ActionOutput```，框架会侦测改输出的类型，并在运行时传入```ActionOutput```实例，活动通过```ActionOutput```实例来输出参数。
1. 活动的输入参数，除了上下文参数和输出参数，其他参数都为活动的输入参数。

在上面的代码中，第一个参数是该活动的输入参数，第二个参数是`IExecutionContext`类型，用它可以获取活动执行上下文的信息，例如是哪个责任者执行的该活动。

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

# 多个活动组合

一个行为可以包含多个活动，多个活动可以组合成树状结构，位于树根的活动是行为的入口，因此该活动的输入类型必须与行为的输入数据类型匹配，所有位于树叶的活动为行为的出口，因此所有树叶的活动的输出数据类型一致。

当多个活动组合时，前一个活动的输出必须与后一个活动的输入保持一致（数量和类型），否则将会抛出运行时异常。
一个活动可以指定多个后续活动，框架提供了API根据前一个活动的输出内容来确定在运行时执行哪个后续活动。

下面的例子展示了如何组合多个活动，这个例子模拟用户登入，验证用户然后输出成功或失败信息：
首先我们定义了一个活动，它需要获取并返回用户的信息，当然这里我们只是硬编码了用户的信息，在真实环境中这些信息应该从用户输入或者网络请求中读入：

```java
@Service
@Action
@Tag("UserLogin")
public class FetchUserInfoAction {

    @ActionDo
    public void execute(AppStartupEvent event, ActionOutput<String> name, ActionOutput<String> password) {
        name.set("Min");
        password.set("password");
    }
}
```

这个```FetchUserInfoAction```的第一个参数是输入参数，不过这个例子用不到它，第二和第三个参数是输出参数，分别用来输出用户名和密码。

然后我们定义了验证用户信息的活动：

```java
@Service
@Action
@Tag("UserLogin")
public class LoginUserAction {

    private static final String USER_NAME   = "Min";
    private static final String USER_PWD    = "password";

    @ActionDo
    public void login(final String name, final String password, ActionOutput<Boolean> isSuccess) {
        boolean matched = USER_NAME.equals(name) && USER_PWD.equals(password);
        isSuccess.set(matched);
    }
}
```

```LoginUserAction```的第一和第二个参数分别是输入用户名和密码，第三个参数是个输出参数，用来输出登入成功或者失败。

然后我们需要定义两个活动，分别对应登入成功以及登入失败的活动：

```java
@Service
@Action
@Tag("UserLogin")
public class LoginSuccessAction {

    @Inject
    protected ILogger _logger;

    @ActionDo
    public void doSuccess(boolean isSuccess) {
        if (isSuccess) {
            this._logger.info("Login user success");
        }
    }
}

@Service
@Action
@Tag("UserLogin")
public class LoginFailedAction {

    @Inject
    protected ILogger _logger;

    @ActionDo
    public void doFailed(boolean isSuccess) {
        if (! isSuccess) {
            this._logger.info("Login user failed");
        }
    }
}
```

为了使得例子简单，我们在这两个活动里只是打印一些日志来表明登入成功或者失败。

最后我们需要将这些活动装配到行为中去：

```java
@Service(autoActive = true)
@Tag("UserLogin")
public class LoginUserApp {

    @Inject
    protected IResponsibleRegistry _respReg;

    @OnActivate
    public void activate() {
        this._respReg.register("Login User")
                .newBehavior("Login User", AppStartupEvent.class, AppStartupEvent.TOPIC)
                .then(FetchUserInfoAction.class)
                .then(LoginUserAction.class, "login")
                .then(LoginFailedAction.class)
                .navigator().moveTo("login")
                .when(attr -> attr.get("isSuccess"))
                .then(LoginSuccessAction.class)
                .build();
    }
}
```

在装配活动的时候需要知道一直有个指针指向当前设置的活动，如果用```then```方法添加活动的时候只是将新增的活动放在当前活动之后，然后指针指向新增的活动，所以当我们需要在一个活动后面增加两个活动的时候，需要移动指针到前一个活动，这里就需要标签，通过标签可以将指针跳转到指定标签的活动上去。

这个例子里面为```LoginUserAction```指定了```login```标签，因此后面可以通过```navigator```方法来获取指针对象并通过```moveTo```方法移动到绑定```login```标签的```LogingUserAction```活动。

当一个活动有多个后续活动的时候，可以通过```when```方法指定执行后续活动的条件，当指定的条件满足了，后续的活动才会被执行。```when```方法接收一个参数，该参数是一个```IAttributed```实例，该对象类似一个```Map```，每个元素都有名称和对应的值，分别是前一个活动的输出的名称和对应的输出值。

在这个例子里面```LoginFailedAction```为默认后续活动，当指定多个后续活动时，必须指定一个默认的后续活动，当其他有条件的后续活动的条件都不满足时，默认的后续活动将会被执行。

```LoginSuccessAction```是非默认后续活动，当```isSuccess```为真的时候才会被调用。

我们来运行这个例子：

```shell
6:57:56.848 [Thread-4] INFO  u.c.internal.FileBasedConfigProvider - Config update system.config -> conf/login-user.yaml
16:57:56.976 [Thread-4] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/login-user.yaml
16:57:57.136 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application.StartupApplication - Application is going to startup...
16:57:57.137 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - UserLoginProfile
16:57:57.214 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application.StartupApplication - The application is launched
16:57:57.215 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.Application - Application startup success.
16:57:57.216 [ForkJoinPool-1-worker-1] INFO  u.t.behavior.LoginSuccessAction - Login user success
```

可以看到最后日志输出了```Login user success```的内容表明```LoginSuccessAction```被调用了。
如果我们将用户名和密码改动一下，再次运行：

```shell
16:59:39.686 [Thread-4] INFO  u.c.internal.FileBasedConfigProvider - Config update system.config -> conf/login-user.yaml
16:59:39.852 [Thread-4] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/login-user.yaml
16:59:40.112 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application.StartupApplication - Application is going to startup...
16:59:40.113 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - UserLoginProfile
16:59:40.165 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application.StartupApplication - The application is launched
16:59:40.165 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.Application - Application startup success.
16:59:40.166 [ForkJoinPool-1-worker-0] INFO  u.t.behavior.LoginFailedAction - Login user failed
```

可以看到```LoginFailedAction```被调用了。

# 事件和行为的绑定

# 行为复用

行为本身也是有输入和输出，因此行为可以作为活动来进行复用的。
