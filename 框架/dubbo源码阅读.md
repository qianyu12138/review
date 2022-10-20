# 容器

## 追溯功能点

选择一个拓展点追溯

> org.apache.dubbo.remoting.transport.netty.NettyServer#NettyServer

追溯调用


> org.apache.dubbo.remoting.transport.netty.NettyTransporter#bind


追溯调用


> org.apache.dubbo.remoting.Transporters#bind(org.apache.dubbo.common.URL, org.apache.dubbo.remoting.ChannelHandler...)


```java
public static RemotingServer bind(URL url, ChannelHandler... handlers) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handlers == null || handlers.length == 0) {
        throw new IllegalArgumentException("handlers == null");
    }
    ChannelHandler handler;
    if (handlers.length == 1) {
        handler = handlers[0];
    } else {
        handler = new ChannelHandlerDispatcher(handlers);
    }
    return getTransporter(url).bind(url, handler);
}

public static Transporter getTransporter(URL url) {
  	return url.getOrDefaultFrameworkModel().getExtensionLoader(Transporter.class).getAdaptiveExtension();
}
```

## 实现分析

> org.apache.dubbo.common.extension.ExtensionDirector#getExtensionLoader

```java
@Override
public <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    checkDestroyed();
  	//private final AtomicBoolean destroyed = new AtomicBoolean(); 并发场景检查容器状态
    if (type == null) {
        throw new IllegalArgumentException("Extension type == null");
    }
    if (!type.isInterface()) {//目标类型必须是interface
        throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
    }
    if (!withExtensionAnnotation(type)) {//目标类型必须标注扩展点标识
        throw new IllegalArgumentException("Extension type (" + type +
            ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
    }

    // 1. find in local cache
    ExtensionLoader<T> loader = (ExtensionLoader<T>) extensionLoadersMap.get(type);//ExtensionDirector是普通对象，每个容器中，每个classloader都是单例的

    ExtensionScope scope = extensionScopeMap.get(type);
    if (scope == null) {
        SPI annotation = type.getAnnotation(SPI.class);
        scope = annotation.scope();
        extensionScopeMap.put(type, scope);
    }//缓存注解中的scope属性

    if (loader == null && scope == ExtensionScope.SELF) {
        // create an instance in self scope
        loader = createExtensionLoader0(type);
    }

    // 2. find in parent
    if (loader == null) {
        if (this.parent != null) {
            loader = this.parent.getExtensionLoader(type);
        }
    }//如果没有则创建一个loader，该处为使用本身的scope

    // 3. create it
    if (loader == null) {
        loader = createExtensionLoader(type);
    }//如果没有则创建一个loader，该处为使用注解中的scope

    return loader;
}
```

> org.apache.dubbo.common.extension.ExtensionLoader#getAdaptiveExtension

```java
public T getAdaptiveExtension() {
  checkDestroyed();
  Object instance = cachedAdaptiveInstance.get();//从缓存中获取
  if (instance == null) {
    if (createAdaptiveInstanceError != null) {
      //private volatile Throwable createAdaptiveInstanceError;并发场景下其他线程创建失败
      throw new IllegalStateException("Failed to create adaptive instance: " +
                                      createAdaptiveInstanceError.toString(),
                                      createAdaptiveInstanceError);
    }

    synchronized (cachedAdaptiveInstance) {
      instance = cachedAdaptiveInstance.get();//双重校验锁，这也是Holder存在的意义，代替对象本身成为锁的目标
      if (instance == null) {
        try {
          instance = createAdaptiveExtension();
          cachedAdaptiveInstance.set(instance);
        } catch (Throwable t) {
          createAdaptiveInstanceError = t;
          throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
        }
      }
    }
  }
  return (T) instance;
}

//非常像spring的bean创建
private T createAdaptiveExtension() {
  try {
    T instance = (T) getAdaptiveExtensionClass().newInstance();
    instance = postProcessBeforeInitialization(instance, null);
    instance = injectExtension(instance);
    instance = postProcessAfterInitialization(instance, null);//在其中也有一些aware的功能
    initExtension(instance);
    return instance;
  } catch (Exception e) {
    throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
  }
}

private Class<?> getAdaptiveExtensionClass() {
  getExtensionClasses();
  if (cachedAdaptiveClass != null) {
    return cachedAdaptiveClass;
  }
  return cachedAdaptiveClass = createAdaptiveExtensionClass();
}

//获取对应的Adaptiv创建
private Class<?> createAdaptiveExtensionClass() {
  // Adaptive Classes' ClassLoader should be the same with Real SPI interface classes' ClassLoader
  ClassLoader classLoader = type.getClassLoader();
  try {
    if (NativeUtils.isNative()) {
      return classLoader.loadClass(type.getName() + "$Adaptive");
    }
  } catch (Throwable ignore) {

  }
  String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
  org.apache.dubbo.common.compiler.Compiler compiler = extensionDirector.getExtensionLoader(
    org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
  return compiler.compile(type, code, classLoader);
}
```