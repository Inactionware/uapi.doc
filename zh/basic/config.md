# 简介

每个应用程序或多或少都会支持通过外部配置来更改应用的行为，UAPI框架也不例外。

UAPI框架通过`Config`注解来注入外部配置。

如果需要配置需要倒入一下依赖：

```groovy
dependencies {
    compileOnly "uapi:uapi.config.apt:${uapi_cornerstone_version}
    compile "uapi:uapi.config:${uapi_cornerstone_version}"
}
```

`uapi.config.apt`中包含了`Config`注解以及在编译时解析该注解的代码，`uapi.config`包含了配置框架的API和相应的处理逻辑。

# 简单配置注入

# 自定配置注入
