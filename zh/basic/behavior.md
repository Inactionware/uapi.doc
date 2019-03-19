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

当多个活动组合的时候，默认情况前一个活动的输出默认和后一个活动的输入相匹配，但是你可以指定任意一个活动的输出作为当前活动的输入，使用```IWired```接口来指定任意前序活动的输出，通过```IBehaviorBuilder::wired```方法来获取```IWired```的实例对象。你还可以直接指定当前活动的输入对象，比如当前活动需要一个字符串作为输入，你可以直接调用```IBehaviorBuilder::then(class, String, Object...)```方法来指定

# 匿名活动

前面的列子都是通过新的类型来定义行为，然后通过活动的类型或者ID来将活动加入到行为中，但是当一个活动的代码非常简单的情况下，这种方式会非常麻烦，框架提供了通过匿名行为的方式来简化活动的定义。

例如我们可以在上面的```SimpleApp```中加入匿名活动：

```java
    @OnActivate
    public void activate() {
        this._respReg.register("A Responsible")
                .newBehavior("Simple Behavior", AppStartupEvent.class, AppStartupEvent.TOPIC)
                .then(SimpleAction.actionId)
                .call(execCtx -> this._logger.info("End execute SimpleAction by {}.", execCtx.behaviorName()))
                .build();
    }
```

这里使用了```call```方法传入了一个Lambda表达式，该表达式包含了匿名活动需要执行的代码。

虽然匿名活动非常简单，但是和传统的活动而言，它不能有输入和输出，不过您可以通过```IExecutionContext```来获取当前行为的输入以及其他活动的输出数据。

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

# 行为跟踪

框架提供API来对行为执行的状况进行追踪，这在调试的时候非常有用。

我们创建一个用于追踪的活动：

```java
@Service
@Action
@Tag("TraceBehavior")
public class TraceAction {

    @Inject
    protected ILogger _logger;

    @ActionDo
    public void execute() {
        this._logger.info("I am TraceAction");
    }
}
```

这个简单的活动就是输出一行日志信息用以表明该活动被调用了。

接下来创建App：

```java
@Service(autoActive = true)
@Tag("TraceBehavior")
public class TraceApp {

    @Inject
    protected IResponsibleRegistry _respReg;

    @Inject
    protected ILogger _logger;

    private final BehaviorExecutingEventHandler execHandler = (execEvent) -> {
        this._logger.info(
                "Action executed: {}\n\tAction inputs: {}\n\tAction outputs: {}",
                execEvent.executingActionId().toString(),
                CollectionHelper.asString(execEvent.currentInputs()),
                CollectionHelper.asString(execEvent.currentOutputs()));
        return null;
    };

    @OnActivate
    public void activate() {
        IResponsible resp = this._respReg.register("Tracer");
        resp.newBehavior("Tracer Behavior", AppStartupEvent.class, AppStartupEvent.TOPIC)
                .then(TraceAction.class)
                .traceable(true)
                .build();
        resp.on(execHandler);
    }
}
```

首先我们用Lambda表达式声明了一个```BehaviorExecutingEventHandler```的实例，逻辑非常简单，就是输出当前被执行的活动的编号，输入和输出。 

要追踪一个行为是很简单的，只需要在创建行为的时候调用```traceable```方法，并传入true即可，然后在该行为所属的```IResponnsible```对象上传入```IBehaviorExecutingHandler```对象，那么当行为执行完每个活动后都会调用该Handler对象。

执行该代码，可以得到：

```shell
13:22:50.434 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config update system.config -> conf/trace-behavior.yaml
13:22:50.475 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/trace-behavior.yaml
13:22:50.514 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application.StartupApplication - Application is going to startup...
13:22:50.514 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - TraceBehaviorProfile
13:22:50.521 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application.StartupApplication - The application is launched
13:22:50.521 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.Application - Application startup success.
13:22:50.523 [ForkJoinPool-1-worker-2] INFO  uapi.tutorial.behavior.TraceAction - I am TraceAction
13:22:50.523 [ForkJoinPool-1-worker-1] INFO  uapi.tutorial.behavior.TraceApp - Action executed: uapi.behavior.internal.Behavior.HeadAction@ACTION
    Action inputs: 
    Action outputs: uapi.app.AppStartupEvent@2e1f2657
13:22:50.523 [ForkJoinPool-1-worker-3] INFO  uapi.tutorial.behavior.TraceApp - Action executed: uapi.tutorial.behavior.TraceAction@ACTION
    Action inputs: 
    Action outputs: 
```

正如代码所写，会在控制台输出每个被执行的活动和活动的输入和输出，这里可以看到有个```HeadAction```特殊活动被执行了，这是一个内部对象，它会在每个活动执行时第一个被执行。

因为每个活动执行后都会调用```Handler```，因此如果```Handler```里面有大量逻辑的话会极大影响行为的执行效率，所以这种情况下应该在```Handler```最后返回一个```BehaviorEvent```，然后在另一个行为事件处理器中处理该耗时操作。

# 事件触发行为

传统的服务端程序中，需要执行许多业务逻辑，大部分逻辑会访问外部资源，例如查询数据库，读写文件，请求外部服务数据等等，通常为了程序易于阅读我们会让线程阻塞直到得到所需的数据才会继续执行，这种情况我们会创建多个线程来使得程序能支持多的并发量，但是所带来的问题就是很多线程都处于阻塞状态，使得资源利用率不高，而且管理多个线程对于系统而言消耗也是非常大的，所以在这种IO密集型的程序中，多线程的阻塞模式是不适合的，取而代之的应该使用无阻塞的模型。

简单的说无阻塞模型就是当遇到请求外部耗时资源的时候，例如读写文件，网络请求等，线程并不等待响应的返回，而是注册回调并继续执行后续逻辑或者释放当前线程，当外部资源响应到达的时候再调用先前注册的回调逻辑。

行为框架对非阻塞的模式提供了支持，因为行为本身可以由事件触发，并且在行为执行过程中以及结束后可以选择抛出特定的事件，则通过事件我们可以把多个相关业务逻辑分开或者并行执行，比如我们需要向网络发送请求，可以在请求发送之后将上下文保存并结束当前行为，然后当响应到达后发出新的事件，由新的事件处理器恢复上下文继续后续的逻辑处理。

下面的例子展示了如何在行为执行过程中抛出事件：

```java
@Service
@Action
@Tag("EventTrigger")
public class EventTriggerSourceAction {

    @Inject
    protected ILogger _logger;

    @ActionDo
    public void execute(final AppStartupEvent event, final IExecutionContext execCtx) {
        this._logger.info("Raise an event from {}", execCtx.responsibleName());
        execCtx.fireEvent(new TriggerEvent(execCtx.responsibleName()));

    }
}
```

首先我们创建了一个活动，这个活动在执行过程中利用上下文对象发出了一个自定义的事件。

```java
@Service
@Action
@Tag("EventTrigger")
public class EventTriggerTargetAction {

    @Inject
    protected ILogger _logger;

    @ActionDo
    public void execute(final TriggerEvent event) {
        this._logger.info("Receive an event from {}", event.sourceName());
    }
}
```

这里创建了一个活动，它的输入时上述活动发出的事件对象，它只是打印日志输出了事件源的信息。

```java
public class TriggerEvent extends BehaviorEvent {

    public static final String TOPIC    = "MyEventTopic";

    public TriggerEvent(String sourceName) {
        super(TOPIC, sourceName);
    }
}
```

事件定义非常简单，你可以在你自己的事件里面定义任何数据，这里要注意事件必须继承```BehaviorEvent```对象，否则无法触发响应的行为。

```java
@Service(autoActive = true)
@Tag("EventTrigger")
public class EventTriggerApp {

    @Inject
    protected IResponsibleRegistry _respReg;

    @Inject
    protected ILogger _logger;

    @OnActivate
    public void activate() {
        IResponsible myResp = this._respReg.register("My Responsible");
        myResp.newBehavior("My Behavior", AppStartupEvent.class, AppStartupEvent.TOPIC)
                .then(EventTriggerSourceAction.class)
                .then(Sleep.class, "Sleep", IntervalTime.parse("1s"))
                .onSuccess((success, ctx) -> { this._logger.info("{} execution done", ctx.behaviorName()); return null; })
                .build();

        IResponsible otherResp = this._respReg.register("My Other Responsible");
        otherResp.newBehavior("Other Behavior", TriggerEvent.class, TriggerEvent.TOPIC)
                .then(EventTriggerTargetAction.class)
                .build();
    }
}
```

这段代码创建了两个责任者，第一个名为```myResp```的责任者创建了一个行为，这个行为包含了先前定义的会发送自定义事件的活动，然后会等待1秒（```Sleep```活动），最后在行为执行成功后输出一行日志。第二个责任者名为```otherResp```，它定义了一个行为，该行为就一个活动用来接收自定的事件。

运行该代码得到下面的输出：

```shell
16:55:20.136 [Thread-4] INFO  u.c.internal.FileBasedConfigProvider - Config update system.config -> conf/event-trigger.yaml
16:55:20.233 [Thread-4] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/event-trigger.yaml
16:55:20.411 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application.StartupApplication - Application is going to startup...
16:55:20.412 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - EventTriggerProfile
16:55:20.436 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application.StartupApplication - The application is launched
16:55:20.436 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.Application - Application startup success.
16:55:20.452 [ForkJoinPool-1-worker-1] INFO  u.t.b.EventTriggerSourceAction - Raise an event from My Responsible
16:55:20.453 [ForkJoinPool-1-worker-0] INFO  u.t.b.EventTriggerTargetAction - Receive an event from My Responsible
16:55:21.455 [ForkJoinPool-1-worker-1] INFO  u.tutorial.behavior.EventTriggerApp - My Behavior execution done
```

通过让第一个行为等待1秒，可以看到两个行为是并行执行的。
