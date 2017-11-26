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

# 服务的注入
服务注入的行为是由服务框架根据服务的依赖关系自动在服务被使用的时候发生的（当autoActivate为false的时候）。
服务框架会首先扫描该服务所有的依赖并检测所有的依赖是否都可用，服务注入的行为发生在所有的依赖都可用的情况下，否则会抛出异常指明服务不可用。
使用`@Inject`注解注入服务。
> Inject注解支持字段注入和方法注入，并且字段或方法不能是`private`,`static`, `final`的，当用于方法注入的时候，该方法必须只有一个输入参数，并且没有返回。

## 基于类型的注入
基于类型的注入是推荐的方式，比如：
```java
@Service
public class Test {
    @Inject
    protected OtherService _otherSvc;
    ...
}

@Service
public class OtherService { ... }
```
当服务框架扫描Test类的时候，会扫描所有的`Inject`注解，并根据字段的声明类型自动推断出Test服务依赖OtherService服务，因此在初始化Test服务的时候自动在服务注册中心中查找OtherService服务并注入到Test服务中去。

## 基于标识的注入
有些时候我们无法使用基于类型的方式注入的时候，我们可以直接使用自定义的服务标识来注入，比如：
```java
@Service
public class Test {
    @Inject("otherSvc")
    protected OtherService _otherSvc;
    ...
}

@Service(ids="otherSvc")
public class OtherService { ... }
```
在这个列子里面Test服务依赖了一个标识为otherSvc的服务，因此服务框架会在服务注册中心查找所有标识为otherSvc的服务并注入到Test服务中去。
> 这里要非常注意注入服务的类型匹配，如果注入的服务与声明服务注入的字段类型不匹配的话会抛出运行时异常，因为只有在运行时才能发现该错误，因此如非必要，请使用基于类型的注入方式。

## 使用服务工厂
在某些情况下我们声明的服务是一个服务工厂，当服务注入发生的时候由该工厂返回真正的服务，这时候我们需要使用服务工厂，例如：
```java
@Service
public class Test {
    @Inject
    protected OtherService _otherSvc;
    ...
}

@Service
public class OtherService implements IServiceFactory<OtherService> {
    @Override
    public OtherService createService(Object serveFor) {
        return new OtherService();
    }
}
```
只要实现了`IServiceFactory`接口，该服务就成为了服务工厂，使用工厂方法`createService`来返回真正的服务，该工厂方法只有一个名为`serveFor`的参数，这个参数指明了返回的服务将被注入到哪个服务中去。

## 通过方法注入服务
使用`Inject`注解不仅可以声明在类字段中注入服务，也可以在方法上声明，指明在注入发生的时候使用该方法来注入，通常如果我们做一些额外的逻辑的时候就可以使用方法注入，例如：
```java
@Service
public class Test {
    private OtherService _otherSvc;

    @Inject
    protected void setService(OtherService svc) {
        this._otherSvc = svc;
        System.out.println("Inject OtherService is successful.");
    }
}

@Service
pubic class OtherService { ... }
```
这里使用方法注入了OtherService，并且在注入完成后在系统控制台上输出了注入成功的信息。
>  使用Inject方法注入时，方法不能是`private`,`static`, `final`的，该方法必须只有一个输入参数，并且没有返回。

## 多服务的注入
当一个服务依赖的服务有多个实例的时候，我们需要使用多服务的注入

### 基于List的注入
默认的情况下多服务注入都是基于`java.util.List`的，例如：
```java
@Service
public class Test {
    @Inject
    protected List<OtherService> _otherSvc;
    ...
}

pubilc interface OtherService { ... }

@Service(OtherService.class)
public class OtherServiceImpl1 implements OtherService { ... }

@Service(OtherService.class)
public class OtherServiceImpl2 implements OtherService { ... }
```
在这个例子中Test服务依赖OtherService，OtherService是一个接口类型，它有两个服务实现，因此服务框架在初始化Test服务的时候会将OtherServiceImpl1和OtherServiceImpl都注入到Test服务中去。

### 基于Map的注入
当注入的服务实现了`IIdentifiable`接口的时候，你可以将服务注入到`java.util.Map`中去，例如：
```java
@Service
public class Test {
    @Inject
    protected Map<String, OtherService> _otherSvc;
}

pubic interface OtherService { ... }

@Service(OtherService.class)
public class OtherServiceImpl1 implements OtherService, IIdentifiable<String> {
    @Override
    String getId() {
        return "1";
    }
}

@Service(OtherService.class)
pubilc class OtherServiceImpl2 implements OtherService, IIdentifiable<String> {
    @Override
    String getId() {
        return "2";
    }
}
```
在这个列子中Test服务依赖OtherService，并且注入的是Map，Map的Key类型是String类型。OtherService的两个实现都实现了IIdentifiable接口，该接口的实现方法都返回了一个字符串类型，因此当服务框架初始化Test服务的时候会将OtherService的两个实现都注入到_otherSvc的Map中去，并分别与1，2着两个Key关联。

# 服务装载器
当服务注入发生的时候，服务框架会调用服务装载器来搜寻和装载服务，在服务框架中内置一个名为`local`的服务装载器，顾名思义该服务装载器用于扫描本地服务注册中心来装载服务的，但是很多情况下我们需要的服务可能来自外部，比如Spring上下文或者来自外部网络，这时候需要使用外部的服务装载器来装载这些服务。
通过实现`IServiceLoader`接口你来加入自定义的服务装载器。