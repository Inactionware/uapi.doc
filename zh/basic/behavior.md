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

# 行为复用

行为本身也是有输入和输出，因此行为可以作为活动来进行复用的。

下面例子里面我们先创建一个活动：

```java
@Service
@Action
@Tag("SayGreeting")
public class GetUserAction {

    @ActionDo
    public void get(
            final AppStartupEvent event,
            final ActionOutput<String> name
    ) {
        name.set("Min");
    }
}
```

这个活动模拟获取用户名，并将作为活动的输出。

然后我们创建一个活动用来产生一个问候语句：

```java
@Service
@Action
@Tag("SayGreeting")
public class MakeGreetingAction {

    @ActionDo
    public void makeGreeting(
            final String name,
            final ActionOutput<String> greeting
    ) {
        String msg = StringHelper.makeString("Hi {}, how are you!", name);
        greeting.set(msg);
    }
}
```

这个活动很简单，把输入的用户名字拼接到问候语中，然后返回该问候语。

最后我们将问候语输出到控制台上：

```java
@Service
@Action
@Tag("SayGreeting")
public class OutGreetingAction {

    @Inject
    protected ILogger _logger;

    @ActionDo
    public void out(final String greeting) {
        this._logger.info(greeting);
    }
}
```

接下来我们展示如何复用行为：

```java
@Service(autoActive = true)
@Tag("SayGreeting")
public class GreetingApp {

    @Inject
    protected IResponsibleRegistry _respReg;

    @OnActivate
    public void activate() {
        IResponsible greeting = this._respReg.register("Greeting Maker");
        IBehavior makeGreeting = greeting.newBehavior("Make Greeting", String.class)
                .then(MakeGreetingAction.class)
                .build();

        greeting.newBehavior("Out Greeting", AppStartupEvent.class, AppStartupEvent.TOPIC)
                .then(GetUserAction.class)
                .then(makeGreeting.getId())
                .then(OutGreetingAction.class)
                .build();
    }
}
```

上面的代码，我们首先创建了一个名为```greeting```的```IResponsible```对象，该对象是用来创建行为的。
然后我们在```greeting```上创建了一个```Make Greeting```的行为，这个行为只有一个活动。

接下来我们创建了一个新的名为```Out Greeting```的行为，这个行为包含了三个动作，其中第二个动作其实就是```Make Greeting```的行为。

执行该程序将获得下面输出：

```shell
13:32:57.974 [Thread-4] INFO  u.c.internal.FileBasedConfigProvider - Config update system.config -> conf/say-greeting.yaml
13:32:58.103 [Thread-4] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/say-greeting.yaml
13:32:58.303 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application.StartupApplication - Application is going to startup...
13:32:58.303 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - SayGreetingProfile
13:32:58.364 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application.StartupApplication - The application is launched
13:32:58.365 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.Application - Application startup success.
13:32:58.366 [ForkJoinPool-1-worker-1] INFO  u.t.behavior.OutGreetingAction - Hi Min, how are you!
```

# 事件和行为的绑定

传统的服务端程序中，需要执行许多业务逻辑，大部分逻辑会访问外部资源，例如查询数据库，读写文件，请求外部服务数据等等，通常为了程序易于阅读我们会让线程阻塞直到得到所需的数据才会继续执行，这种情况我们会创建多个线程来使得程序能支持多的并发量，但是所带来的问题就是很多线程都处于阻塞状态，使得资源利用率不高，而且管理多个线程对于系统而言消耗也是非常大的，所以在这种IO密集型的程序中，多线程的阻塞模式是不适合的，取而代之的应该使用无阻塞的模型。

行为框架给非阻塞的模式提供了支持。

行为本身可以由事件触发，并且在行为结束时可以抛出事件或者在行为执行过程中通过上下文的对象抛出特定的事件。

# 行为跟踪

框架提供API来对行为执行的状况进行追踪，这在调试的时候非常有用。
