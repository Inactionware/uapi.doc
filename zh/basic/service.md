# 服务声明
服务在使用前必须声明，使用`@Service`来声明一个类是受UAPI框架管理的服务，比如：
```java
@Service
public class Test {
    ...
}
```
这样当UAPI服务框架启动后会自动创建该服务，并管理该服务与其他服务之间的依赖。
> 注意，一旦类声明为服务后，该类不能为`private`以及`final`的，否则在编译时会抛出异常。

## 服务标识
每个服务都必须有至少一个标识，通过该标识服务框架就可以明确服务之间的依赖关系。
该标识默认是使用该类的完整类名作为服务标识，你可以自己指定任意字符串作为服务标识。

## 服务类型
每个服务都会有至少一个的类型，默认情况下，服务框架会使用类的完整类名作为服务的类型，你可以指定一个或多个类型作为该服务的类型，比如类实现了多个接口的时候，你可以指定多个接口类型为该服务的类型，这样服务框架可以使用这些接口类型将该服务注入多个依赖这些接口的其他服务中去。
> 注意，服务类型在服务框架中最终会被转换成服务标识。

## 服务自动激活
当系统启动的时候，服务框架会扫描所有的服务，并登记这些服务以及其依赖关系到服务注册中心，默认情况下只有当服务被使用的时候才会对该服务进行初始化以及激活使其对外提供服务。但是在某些情况下一些服务需要在系统启动后被立即激活，例如某个服务是用来侦听网络接口。
使用`autoActive`来指定某个服务需要在所有依赖满足的情况下自动激活。

```java
@Service(autoActive = true)
@Tag("AutoActivated")
public class AutoActivatedService {

    @Inject
    protected ILogger _logger;

    @OnActivate
    public void activated() {
        this._logger.info("I am an AutoActivatedService...");
    }
}
```

UAPI框架启动该服务的时候会自动激活该服务：

```shell
13:13:58.406 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config update system.config -> conf/auto-activated-service.yaml
13:13:58.449 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/auto-activated-service.yaml
13:13:58.470 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - Application is going to startup...
13:13:58.470 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - AutoActivatedService
13:13:58.473 [ForkJoinPool.commonPool-worker-1] INFO  u.t.service.AutoActivatedService - I am an AutoActivatedService...
13:13:58.473 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - The application is launched
13:13:58.473 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.Application - Application startup success.
13:13:58.473 [ForkJoinPool-1-worker-1] WARN  uapi.event.internal.EventBus - There are no event handler for event topic - ApplicationStartup
```

从日志中可以看到AutoActivatedService输出了I am an AutoActivatedService。

# 服务的注入
服务注入的行为是由服务框架根据服务的依赖关系自动在服务被使用的时候发生的（当autoActivate为false的时候）。
服务框架会首先扫描该服务所有的依赖并检测所有的依赖是否都可用，服务注入的行为发生在所有的依赖都可用的情况下，否则会抛出异常指明服务不可用。
使用`@Inject`注解注入服务。
> Inject注解支持字段注入和方法注入，并且字段或方法不能是`private`,`static`, `final`的，当用于方法注入的时候，该方法必须只有一个输入参数，并且没有返回。

## 基于类型的注入
基于类型的注入是推荐的方式，比如：
```java
@Service(autoActive = true)
@Tag("TypeBasedInjection")
public class TypeBasedInjectionService {

    @Inject
    protected ILogger _logger;

    @Inject
    protected TypedService _svc;

    @OnActivate
    public void activate() {
        if (this._svc != null) {
            this._logger.info("The TypedService is injected!");
        } else {
            this._logger.error("The TypedService is NOT injected!");
        }
    }
}

@Service
@Tag("TypeBasedInjection")
public class TypedService { }
```
当服务框架扫描`TypeBasedInjectionService`类的时候，会扫描所有的`Inject`注解，并根据字段的声明类型自动推断出该服务依赖`TypedService`服务，因此在初始化`TypeBasedInjectionService`服务的时候自动在服务注册中心中查找`TypedService`服务并注入到`TypeBasedInjectionService`服务中去。

启动该服务的app时：

```shel
13:30:58.681 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config update system.config -> conf/type-based-injection.yaml
13:30:58.730 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/type-based-injection.yaml
13:30:58.766 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - Application is going to startup...
13:30:58.767 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - TypeBasedInjection
13:30:58.772 [ForkJoinPool.commonPool-worker-1] INFO  u.t.s.TypeBasedInjectionService - The TypedService is injected!
13:30:58.773 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - The application is launched
13:30:58.773 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.Application - Application startup success.
13:30:58.773 [ForkJoinPool-1-worker-1] WARN  uapi.event.internal.EventBus - There are no event handler for event topic - ApplicationStartup
```

通过日志可以看到在应用启动的时候，`TypeBadedInjectionService`打印出`The TypedService is injected!`的信息。

## 基于标识的注入
有些时候我们无法使用基于类型的方式注入的时候，我们可以直接使用自定义的服务标识来注入，比如：
```java
@Service(autoActive = true)
@Tag("IdBasedInjection")
public class IdBasedInjectionService {

    @Inject
    protected ILogger _logger;

    @Inject(value = "MyService")
    protected IdentifiedService _svc;

    @OnActivate
    protected void activate() {
        if (this._svc != null) {
            this._logger.info("The IdentifiedService is injected!");
        } else {
            this._logger.error("The IdentifiedService is NOT injected!");
        }
    }
}

@Service(ids = {"MyService"})
@Tag("IdBasedInjection")
public class IdentifiedService { }
```
在这个列子里面`IdBasedInjectionService`服务依赖了一个标识为`MyService`的服务，因此服务框架会在服务注册中心查找所有标识为`MyService`的服务并注入到`IdBasedInjectionService`服务中去。
> 这里要非常注意注入服务的类型匹配，如果注入的服务与声明服务注入的字段类型不匹配的话会抛出运行时异常，因为只有在运行时才能发现该错误，因此如非必要，请使用基于类型的注入方式。

启动该服务所属的app，将会看到以下日志：

```shel
13:59:03.376 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config update system.config -> conf/id-based-injection.yaml
13:59:03.421 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/id-based-injection.yaml
13:59:03.444 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - Application is going to startup...
13:59:03.444 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - IdBasedInjection
13:59:03.448 [ForkJoinPool.commonPool-worker-1] INFO  u.t.service.IdBasedInjectionService - The IdentifiedService is injected!
13:59:03.448 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - The application is launched
13:59:03.448 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.Application - Application startup success.
13:59:03.448 [ForkJoinPool-1-worker-1] WARN  uapi.event.internal.EventBus - There are no event handler for event topic - ApplicationStartup
```

通过日志可以看到，当应用启动时，`IdBasedInjectionService`打印出`The IdentifiedService is injected!`的信息。

## 基于工厂注入（建议使用原型注入）

在某些情况下我们声明的服务是一个服务工厂，当服务注入发生的时候由该工厂返回真正的服务，这时候我们需要使用服务工厂，例如：
```java
@Service(autoActive = true)
@Tag("FactoryBasedInjection")
public class FactoryBasedInjectionService {

    @Inject
    protected ILogger _logger;

    @Inject
    protected IAService _svc;

    @OnActivate
    public void activate() {
        if (this._svc != null) {
            this._logger.info("The AService is injected!");
        } else {
            this._logger.error("The AService is NOT injected!");
        }
    }
}

public interface IAService { }

@Service
@Tag("FactoryBasedInjection")
public class AServiceFactory implements IServiceFactory<IAService> {

    @Override
    public IAService createService(Object serveFor) {
        return new IAService() { };
    }
}
```
只要实现了`IServiceFactory`接口，该服务就成为了服务工厂，使用工厂方法`createService`来返回真正的服务，该工厂方法只有一个名为`serveFor`的参数，这个参数指明了返回的服务将被注入到哪个服务中去。

启动该服务所属的应用时，将会看到以下日志：

```shell
14:11:18.179 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config update system.config -> conf/factory-based-injection.yaml
14:11:18.248 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/factory-based-injection.yaml
14:11:18.290 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - Application is going to startup...
14:11:18.290 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - FactoryBasedInjection
14:11:18.296 [ForkJoinPool.commonPool-worker-1] INFO  u.t.s.FactoryBasedInjectionService - The AService is injected!
14:11:18.296 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - The application is launched
14:11:18.296 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.Application - Application startup success.
14:11:18.296 [ForkJoinPool-1-worker-1] WARN  uapi.event.internal.EventBus - There are no event handler for event topic - ApplicationStartup
```

通过日志我们可以看到`FactoryBasedInjectionService`在启动中打印出`The AService is injected!`的信息。

## 基于原型注入

原型类似于工厂注入，所区别的是原型注入无需创建一个服务工厂，每当注入发生的时候，框架会创建该原型服务的对象。

基于原型注入的方式要优于基于工厂注入，不仅仅相比基于工厂注入有更少的代码量，而且基于原型注入允许为每个原型实例提供不同的参数。

例子：

```java
@Service(autoActive = true)
@Tag("PrototypeBasedInjection")
public class PrototypeBasedInjectionService {

    @Inject
    protected ILogger _logger;

    @Inject
    protected PrototypeService _protoSvc;

    @OnActivate
    public void activate() {
        if (this._protoSvc != null) {
            this._logger.info("The prototype service was injected.");
        } else {
            this._logger.error("The prototype service was not injected");
        }
    }
}

@Service(type = ServiceType.Prototype)
@Tag("PrototypeBasedInjection")
public class PrototypeService { }
```

`PrototypeService` 是一个原型服务，`PrototypeBasedInjectionService` 是一个依赖原型服务的服务，启动该服务所属的应用，可以得到以下日志：

```shell
17:11:35.590 [ForkJoinPool.commonPool-worker-3] INFO  u.c.internal.FileBasedConfigProvider - Config update system.config -> conf/prototype-based-injection.yaml
17:11:35.623 [ForkJoinPool.commonPool-worker-3] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/prototype-based-injection.yaml
17:11:35.656 [ForkJoinPool-1-worker-3] INFO  u.a.i.Application.StartupApplication - Application is going to startup...
17:11:35.657 [ForkJoinPool-1-worker-3] INFO  uapi.app.internal.ProfileManager - Active profile is - PrototypeBasedInjection
17:11:35.663 [ForkJoinPool.commonPool-worker-3] INFO  u.t.s.PrototypeBasedInjectionService - The prototype service was injected.
17:11:35.663 [ForkJoinPool-1-worker-3] INFO  u.a.i.Application.StartupApplication - The application is launched
17:11:35.663 [ForkJoinPool-1-worker-3] INFO  uapi.app.internal.Application - Application startup success.
17:11:35.663 [ForkJoinPool-1-worker-3] WARN  uapi.event.internal.EventBus - There are no event handler for event topic - ApplicationStartup
```

可以看到日志输出`The prototype service was injected`的信息表明了该原型实例已经注入到了`PrototypeBasedInjectionService`中了。

基于原型的注入方式还可以指定`Attribute`来创建原型实例，例如

```java
@Service(
        type = ServiceType.Prototype,
        value = IHttpListener.class)
public class HttpListener implements IHttpListener {

    @Attribute(HttpAttributes.HOST)
    protected String _host;

    @Attribute(HttpAttributes.PORT)
    protected int _port;
  
    ...
}
```

`HttpListener`是一个负责侦听HTTP请求的原型服务，当初始化的时候它需要侦听的地址和端口，这里我们用`@Attribute`注解在`_host`和`_port`两个字端上，表明初始化该原型实例的时候需要提供地址和端口属性。

当通过服务注册器(IRegistry)来获取原型服务实例的时候要提供原型必须的属性，例如：

```java
@Service
@Action
public class ListenHttpPort {

    public static final ActionIdentify actionId = ActionIdentify.toActionId(ListenHttpPort.class);

    @Config(path="server.host")
    protected String _host;

    @Config(path="server.port")
    protected int _port;

    @Inject
    protected IRegistry _registry;

    @ActionDo
    public void run(Object input) {
        Map<String, Object> attrs = new HashMap<>();
        attrs.put(HttpAttributes.HOST, this._host);
        attrs.put(HttpAttributes.PORT, this._port);
        attrs.put(HttpAttributes.EVENT_SOURCE, "APIServer");
        IHttpListener httpListener = this._registry.findService(IHttpListener.class, attrs);
        httpListener.startUp();
    }
}
```

在`ListenHttpPort`服务中，我们通过配置获取了要侦听的地址和端口，然后初始化属性Map，将该属性Map作为参数从`IRegistry`中获取新的原型服务实例。

## 通过方法注入服务
使用`Inject`注解不仅可以声明在类字段中注入服务，也可以在方法上声明，指明在注入发生的时候使用该方法来注入，通常如果我们做一些额外的逻辑的时候就可以使用方法注入，例如：
```java
@Service(autoActive = true)
@Tag("MethodBasedInjection")
public class MethodBasedInjectionService {

    @Inject
    protected ILogger _logger;

    private TypedService _svc;

    @Inject
    protected void setService(TypedService service) {
        this._svc = service;
    }

    @OnActivate
    public void activate() {
        if (this._svc != null) {
            this._logger.info("The service is injected by method!");
        } else {
            this._logger.error("The service is NOT injected!");
        }
    }
}
```
这里使用`setService`方法注入了`TypedService`，并且在注入完成后在系统控制台上输出了注入成功的信息。

运行该服务所属的应用，你会得到如下输出：

```shell
17:41:42.734 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config update system.config -> conf/method-based-injection.yaml
17:41:42.781 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/method-based-injection.yaml
17:41:42.815 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - Application is going to startup...
17:41:42.815 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - MethodBasedInjection
The TypedService is injected.
17:41:42.822 [ForkJoinPool.commonPool-worker-1] INFO  u.t.s.MethodBasedInjectionService - The service is injected by method!
17:41:42.822 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - The application is launched
17:41:42.822 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.Application - Application startup success.
17:41:42.823 [ForkJoinPool-1-worker-1] WARN  uapi.event.internal.EventBus - There are no event handler for event topic - ApplicationStartup
```

可以看到`The TypedService in injected.`被输出到了控制台。

>  使用Inject方法注入时，方法不能是`private`,`static`, `final`的，该方法必须只有一个输入参数，并且没有返回。
>
>  在该方法内部不能使用其他用@Inject注入的服务，因为在这个是这些服务可能尚未注入。

## 多服务的注入
当一个服务依赖的服务有多个实例的时候，我们需要使用多服务的注入

### 基于List的注入
默认的情况下多服务注入都是基于`java.util.List`的，例如：
```java
@Service(autoActive = true)
@Tag("ListBasedInjection")
public class ListBasedInjectionService {

    @Inject
    protected ILogger _logger;

    @Inject
    protected List<IListItem> _items = new ArrayList<>();

    @OnActivate
    public void activate() {
        if (this._items.size() == 2) {
            this._logger.info("The service is injected into the list!");
        } else {
            this._logger.error("The server is NOT injected into the list!");
        }
    }
}

public interface IListItem { }

@Service(IListItem.class)
@Tag("ListBasedInjection")
public class ListItemA implements IListItem { }

@Service(IListItem.class)
@Tag("ListBasedInjection")
public class ListItemB implements IListItem { }
```
在这个例子中`ListBasedInjectionService`服务依赖`ListItem`，`IListItem`是一个接口类型，它有两个服务实现，因此服务框架在初始化Test服务的时候会将`ListItemA`和`ListItemB`都注入到`ListBasedInjectionService`服务中去。

运行该服务所属的应用，你会得到以下输出：

```shell
18:03:56.125 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config update system.config -> conf/list-based-injection.yaml
18:03:56.173 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/list-based-injection.yaml
18:03:56.196 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - Application is going to startup...
18:03:56.196 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - ListBasedInjection
18:03:56.200 [ForkJoinPool.commonPool-worker-1] INFO  u.t.s.ListBasedInjectionService - The service is injected into the list!
18:03:56.201 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - The application is launched
18:03:56.201 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.Application - Application startup success.
18:03:56.201 [ForkJoinPool-1-worker-1] WARN  uapi.event.internal.EventBus - There are no event handler for event topic - ApplicationStartup
```

你会在控制台中看到`The service is injected into the list!`的信息。

> 注意，基于List注入的字段必须在声明的时候初始化，否则在运行时会抛出空指针异常。
>
> 另外，如果注入服务是基于接口的，那么服务类型上的@Server声明必须标注为以接口类型注册，例如上例中ListItemA和ListItemB以IListItem类型注册的。

### 基于Map的注入
当注入的服务实现了`IIdentifiable`接口的时候，你可以将服务注入到`java.util.Map`中去，例如：
```java
@Service(autoActive = true)
@Tag("MapBasedInjection")
public class MapBasedInjectionService {

    @Inject
    protected ILogger _logger;

    @Inject
    protected Map<String, IMapItem> _items = new HashMap<>();

    @OnActivate
    public void activate() {
        if (this._items.size() == 2) {
            this._logger.info("The services are injected into the map!");
        } else {
            this._logger.error("The services are NOT injected into the map!");
        }
    }
}

public interface IMapItem extends IIdentifiable<String> { }

@Service(IMapItem.class)
@Tag("MapBasedInjection")
public class MapItemA implements IMapItem {

    @Override
    public String getId() {
        return "MapItemA";
    }
}

@Service(IMapItem.class)
@Tag("MapBasedInjection")
public class MapItemB implements IMapItem {

    @Override
    public String getId() {
        return "MapItemB";
    }
}
```
在这个列子中`MapBasedInjectionService`服务依赖`IMapItem`，并且注入的是`Map`，`Map`的`Key`类型是`String`类型。`IMapItem`的两个实现都实现了`IIdentifiable`接口，该接口的实现方法都返回了一个字符串类型，因此当服务框架初始化`MapBasedInjectionService`服务的时候会将`IMapItem`的两个实现都注入到`_items`的`Map`中去，并分别与`MapItemA`，`MapItemB`着两个`Key`关联。

> 注意，Map的value类型必须实现IIdentifiable接口，该接口的模版类型应当与Map的key类型一致，否则将在编译时抛出异常。

运行该服务所属的应用，输出如下：

```shell
18:58:39.065 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config update system.config -> conf/map-based-injection.yaml
18:58:39.107 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/map-based-injection.yaml
18:58:39.127 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - Application is going to startup...
18:58:39.127 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - MapBasedInjection
18:58:39.131 [ForkJoinPool.commonPool-worker-1] INFO  u.t.service.MapBasedInjectionService - The services are injected into the map!
18:58:39.131 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - The application is launched
18:58:39.131 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.Application - Application startup success.
18:58:39.131 [ForkJoinPool-1-worker-1] WARN  uapi.event.internal.EventBus - There are no event handler for event topic - ApplicationStartup
```

可以从控制台输出看到`MapBasedInjectionService`打印了`The services are injected into the map!`信息。

# 服务装载器

当服务注入发生的时候，服务框架会调用服务装载器来搜寻和装载服务，在服务框架中内置一个名为`local`的服务装载器，顾名思义该服务装载器用于扫描本地服务注册中心来装载服务的，但是很多情况下我们需要的服务可能来自外部，比如Spring上下文或者来自外部网络，这时候需要使用外部的服务装载器来装载这些服务。
通过实现`IServiceLoader`接口你来加入自定义的服务装载器。