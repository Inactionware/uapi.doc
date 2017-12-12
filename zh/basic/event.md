# 简介

UAPI框架的核心是基于事件驱动的行为框架，虽然在使用行为框架的时候，无需直接使用事件的API，但是理解事件的API有助于理解整个行为框架的工作模式。

# 定义事件

所有的事件必须实现`IEvent`接口：

```java
package uapi.event;

/**
 * Event interface
 */
public interface IEvent {

    /**
     * Return event topic
     *
     * @return  The event topic
     */
    String topic();
}
```

该接口只有一个方法`topic()`，每个事件通过topic来标识。



# 事件处理器

当某个事件发生时，事件处理器用于处理该事件：

```java
package uapi.event;

import uapi.UapiException;

/**
 * A handler for specific event
 */
public interface IEventHandler<T extends IEvent> {

    /**
     * The topic of event which can be handled by this handler
     *
     * @return  The event handler
     */
    String topic();

    /**
     * Handle event
     *
     * @param   event
     *          The event
     * @throws  UapiException
     *          Handle event failed
     */
    void handle(T event) throws UapiException;
}
```

方法`topic`标识了该事件处理函数所能处理的事件的topic，`handle`方法包含了处理事件的代码逻辑。

事件处理器必须注册才能使用，通过`IEventBus`来注册事件处理器：

```java
{
    @Inject
    protected IEventBus _eventBus;
    
    public void registerEventHandler(IEventHandler eventHandler) {
      this._eventBus.register(eventHandler);
    }
}
```

同样，可以通过`IEventBus`的`unregister`来注销事件处理器。



# 发布事件

可以通过`IEventBus`的`fire`方法来发布指定的事件：

```java
@Service
public class Test {
  
    @Inject
    protected IEventBus _eventBus;
  
    public void fireEvent(MyEvent event) {
        this._eventBus.fire("MyEvent");
    }
}
```

这里发布了一个名为MyEvent的事件，该事件没有任何携带属性，如果需要携带自定义属性，可以定义自己的事件类型，并通过`fire`方法发布自定义事件。

默认情况下事件是按异步方式处理，即事件发布后`fire`方法会立即返回，并不会等待事件处理结束才返回。在`IEventBus`的`fire`方法的某些重载版本中可以指定一个`syncable`的参数，使用该参数可以让`fire`方法阻塞直到事件被处理完成。