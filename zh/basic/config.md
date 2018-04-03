# 简介

每个应用程序或多或少都会支持通过外部配置来更改应用的行为，UAPI框架也不例外。UAPI框架的配置是基于注解的。

UAPI框架通过`Config`注解来注入外部配置。

使用配置前需要导入以下依赖：

```groovy
dependencies {
    compileOnly "uapi:uapi.config.apt:${uapi_cornerstone_version}
    compile "uapi:uapi.config:${uapi_cornerstone_version}"
}
```

`uapi.config.apt`中包含了`Config`注解以及在编译时解析该注解的代码，`uapi.config`包含了配置框架的API和相应的处理逻辑。

> 配置框架默认支持基于文件的配置，需要在启动参数中加入`-config=<config file path>`来告诉框架所使用的配置文件的位置。

# 配置注解

UAPI的配置框架是通过注解类声明的，并且只能声明在类字段上，通过`@Config`来声明该字段的初始内容来自于配置。

`@Config`注解有三个参数：

* `path`指明了该配置所在的路径，按照约定，所有的配置都是用点`.`分割的路径组成。
* `parser`指明了该配置是由哪个解析器来解析的。
* `optional`指明了该配置是否是可选的，如果是必选的那么包含改配置的类必须在该配置注入后才能激活。

# 简单配置注入

简单配置指的配置的类型为基本类型，配置框架自动地将简单配置映射成程序中声明的基本类型。

```java
@Service(autoActive = true)
@Tag("SimpleConfig")
public class SimpleConfigService {

    @Config(path = "cfg.string-value")
    protected String _stringValue;

    @Config(path = "cfg.int-value")
    protected int _intValue;

    @Config(path = "cfg.long-value")
    protected long _longValue;

    @Config(path = "cfg.float-value")
    protected float _floatValue;

    @Config(path = "cfg.double-value")
    protected double _doubleValue;

    @Config(path = "cfg.boolean-value")
    protected boolean _booleanValue;

    @Inject
    protected ILogger _logger;

    @OnActivate
    public void activate() {
        this._logger.info("Config value:");
        this._logger.info("-> String Value is: {}", this._stringValue);
        this._logger.info("-> Integer Value is: {}", this._intValue);
        this._logger.info("-> Long Value is: {}", this._longValue);
        this._logger.info("-> Float Value is: {}", this._floatValue);
        this._logger.info("-> Double Value is: {}", this._doubleValue);
        this._logger.info("-> Boolean Value is: {}", this._booleanValue);
    }
}
```

可以看到对于简单配置而言，`parser`注解参数可以忽略，配置框架可以找到相应的解析器来解析该配置。

配置文件：

```yaml
app:
  active-profile: SimpleConfigAppProfile

profiles:
  - name: SimpleConfigAppProfile
    model: include
    matching: satisfy-any
    tags:
      - SimpleConfig

cfg:
  string-value: TestString
  int-value: 2
  long-value: 300
  float-value: 1.1
  double-value: 23.2
  boolean-value: false
```

对于配置而言，输入的永远是字符串，配置框架根据该配置注入的字段类型找到对应的解析器，解析器会将字符串转成成目标字段的类型，如果无法转换则异常会被抛出。

# 自定配置注入

除了基本的简单类型的配置，很多时候我们需要复杂的配置，比如注入一个类作为配置，这个时候我们需要自定义配置解析器来做配置的解析。

我们需要自定义的配置如下：

```yam
cfg:
  users:
    - name: Mike
      addresses:
        office: Office address
        home: Home address
```

自定义的配置路径是在`cfg.users`下面。

再创建两个简单的类来保存配置信息：

```java
public class User {

    public String name;
    public Address address;

    public String toString() {
        return StringHelper.makeString("name: {}, address: {}", name, address);
    }
}

public class Address {

    public String office;
    public String home;

    public String toString() {
        return StringHelper.makeString("office->{}, home->{}", office, home);
    }
}
```

然后再创建自定义配置解析器：

```java
@Tag("CustomizedConfig")
@Service(IConfigValueParser.class)
public class UserParser implements IConfigValueParser {

    private static final String[] supportedTypesIn  = new String[] { List.class.getCanonicalName() };
    private static final String[] supportedTypesOut = new String[] { List.class.getCanonicalName() };

    @Override
    public boolean isSupport(String inType, String outType) {
        return CollectionHelper.isContains(supportedTypesIn, inType) && CollectionHelper.isContains(supportedTypesOut, outType);
    }

    @Override
    public String getName() {
        return UserParser.class.getCanonicalName();
    }

    @Override
    public List<User> parse(Object value) {
        List<Map> usersCfg = (List) value;
        List<User> users = new ArrayList<>();
        Looper.on(usersCfg).foreach(userCfg -> {
            String name = userCfg.get("name").toString();
            Map addresses = (Map) userCfg.get("addresses");
            String addrOffice = addresses.get("office").toString();
            String addrHome = addresses.get("home").toString();
            User user = new User();
            user.name = name;
            user.address = new Address();
            user.address.office = addrOffice;
            user.address.home = addrHome;
            users.add(user);
        });
        return users;
    }
}
```

自定义配置类解析器需要实现`IConfiguValueParser`接口，当然用`@Service`声明的注解时必须告诉框架以`IConfigValueParser`类型来注册，否则该解析器无法被使用。

`isSupport`方法是框架用来检测配置输入的类型和最终配置注入的字段是否被该解析器所支持。

`parse`方法是解析配置的主要逻辑，在这个例子里面， `users`的配置下面可以配置多个`user`，所以它的类型是`List`，其中每项包括了`name`和`addresses`属性，而`addresses`是一个`Map`，包含了`office`和`home`两个属性。

有了配置类和解析类，我们可以用它来注入服务中去了：

```java
@Tag("CustomizedConfig")
@Service(autoActive = true)
public class CustomizedConfigService {

    @Config(path="cfg.users", parser=UserParser.class)
    protected List<User> _users;

    @Inject
    protected ILogger _logger;

    @OnActivate
    public void onActivate() {
        this._logger.info("The user is {}", this._users);
    }
}
```

用`@Config`注解来注入配置，当然这里还要告诉框架使用哪个解析器来解析响应的配置。

启动该应用，将会得到以下的输出：

```shell
16:25:16.646 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config update system.config -> conf/customized-config.yaml
16:25:16.683 [ForkJoinPool.commonPool-worker-1] INFO  u.c.internal.FileBasedConfigProvider - Config path is conf/customized-config.yaml
16:25:16.714 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - Application is going to startup...
16:25:16.714 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.ProfileManager - Active profile is - CustomizedConfigAppProfile
16:25:16.714 [ForkJoinPool.commonPool-worker-1] INFO  u.t.config.CustomizedConfigService - The user is [name: Mike, address: office->Office address, home->Home address]
16:25:16.714 [ForkJoinPool-1-worker-1] INFO  u.a.i.Application$StartupApplication - The application is launched
16:25:16.714 [ForkJoinPool-1-worker-1] INFO  uapi.app.internal.Application - Application startup success.
```

可以看到当该服务激活的时候，相应的配置也已经注入到指定的字段中去了。