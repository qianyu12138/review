# SpringMVC

![Springmvc](springmvc.jpg)

# Spring

## 生命周期

创建：

- 单实例-容器创建时
- 多实例-每次获取时

构造器->初始化

销毁：

- 单实例-容器销毁时

- 多实例-不销毁（不管理）

初始化方法和销毁方法：除了在配置里指定方法名，也可以实现InitializingBean和DisposableBean并实现方法，或JSR250的相关注解，这些都是对应的BeanPostProcessor做的。



##### BeanPostProcessor原理

populateBean(beanName, mbd, instanceWrapper);给bean进行属性赋值

initializeBean

{

applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);

invokeInitMethods(beanName, wrappedBean, mbd);执行自定义初始化

applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);

}


遍历得到容器中所有的BeanPostProcessor；挨个执行beforeInitialization，
一但返回null，跳出for循环，不会执行后面的BeanPostProcessor.postProcessorsBeforeInitialization



##### Spring底层对 BeanPostProcessor 的使用；

bean赋值，注入其他组件(一些xxxAware，常用的ApplicationContextAware)，@Autowired（AutowiredAnnotationBeanPostProcessor），生命周期注解功能，@Async,xxx BeanPostProcessor;

## 三级缓存

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
	
	/** 一级缓存 Cache of singleton objects: bean name --> bean instance */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);

	/** 三级缓存 Cache of singleton factories: bean name --> ObjectFactory */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);

	/** 二级缓存 Cache of early singleton objects: bean name --> bean instance */
	private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);
  
  //缓存查找bean  如果1级没有，从2级获取,也没有,从3级创建放入2级
        protected Object getSingleton(String beanName, boolean allowEarlyReference) {
            Object singletonObject = this.singletonObjects.get(beanName); //1级
            if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
                synchronized (this.singletonObjects) {
                    singletonObject = this.earlySingletonObjects.get(beanName); //2级
                    if (singletonObject == null && allowEarlyReference) {
                        //3级缓存  在doCreateBean中创建了bean的实例后，封装ObjectFactory放入缓存的
                        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                        if (singletonFactory != null) {
                            //创建未赋值的bean
                            singletonObject = singletonFactory.getObject();
                            //放入到二级缓存
                            this.earlySingletonObjects.put(beanName, singletonObject);
                            //从三级缓存删除
                            this.singletonFactories.remove(beanName);
                        }
                    }
                }
            }
            return singletonObject;
        }
```

```java
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "'beanName' must not be null");
		synchronized (this.singletonObjects) {
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {//如果一级缓存没有
				if (this.singletonsCurrentlyInDestruction) {//是否允许创建单例
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
				}
				beforeSingletonCreation(beanName);//将当前bean标记为创建中
				boolean newSingleton = false;
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<Exception>();
				}
				try {
					singletonObject = singletonFactory.getObject();//执行doCreateBean
					newSingleton = true;
				}
				catch (IllegalStateException ex) {
					// Has the singleton object implicitly appeared in the meantime ->
					// if yes, proceed with it since the exception indicates that state.
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						throw ex;
					}
				}
				catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					afterSingletonCreation(beanName);
				}
				if (newSingleton) {
					addSingleton(beanName, singletonObject);
				}
			}
			return (singletonObject != NULL_OBJECT ? singletonObject : null);
		}
	}
```

```java
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
			throws BeanCreationException {
```

1.不支持循环依赖情况下，只有一级缓存生效，二三级缓存用不到
2.二三级缓存就是为了解决循环依赖，且之所以是二三级缓存而不是二级缓存，主要是可以解决循环依赖对象需要提前被aop代理，以及如果没有循环依赖，早期的bean也不会真正暴露，不用执行无用的代理过程，也不用重复执行代理过程。

根据以上步骤可以看出bean初始化是一个相当复杂的过程，假如初始化A bean时，发现A bean依赖B bean,即A初始化执行到了第3步填充属性，需要注入B bean，此时B还没有初始化，则需要暂停A，先去初始化B，那么此时new出来的A对象放哪里，**直接放在容器Map里显然不合适**，半残品怎么能用，所以需要提供一个可以标记**创建中**bean(A)的Map，可以提前暴露正在创建的bean供其他bean依赖，而如果初始化A所依赖的bean B时，发现B也需要注入一个A的依赖(即发生循环依赖)，则B可以从创建中的beanMap中直接获取A对象（创建中）注入A，然后完成B的初始化，返回给正在注入属性的A，最终A也完成初始化，皆大欢喜。

如果配置不允许循环依赖，则上述缓存就用不到了，A 依赖B，就是创建B，B依赖C就去创建C，创建完了逐级返回就行，所以，一级缓存之后的其他缓存(二三级缓存)就是为了解决循环依赖！而配置支持循环依赖后，就一定要解决循环依赖吗？肯定不是！循环依赖在实际应用中也有，但不会太多，简单的应用场景是： controller注入service，service注入mapper，只有复杂的业务，可能service互相引用，有可能出现循环依赖，所以为了出现循环依赖才去解决，不出现就不解决，虽然支持循环依赖，但是只有在出现循环依赖时才真正暴露早期对象，否则只暴露个获取bean的方法，并没有真正暴露bean，因为这个方法不会被执行到，这块的实现就是三级缓存（singletonFactories），只缓存了一个单例bean工厂。

这个bean工厂不仅可以暴露早期bean还可以暴露代理bean，如果存在aop代理，则依赖的应该是代理对象，而不是原始的bean。而暴露原始bean是在**单例bean初始化**的第2步，填充属性第3步，生成代理对象第4步，这就矛盾了，A依赖到B并去解决B依赖时，要去初始化B，然后B又回来依赖A，而此时A还没有执行代理的过程，所以，需要在填充属性前就生成A的代理并暴露出去，第2步时机就刚刚好。

三级缓存的bean工厂getObject方式，实际执行的是getEarlyBeanReference，如果对象需要被代理(存在beanPostProcessors -> SmartInstantiationAwareBeanPostProcessor)，则**提前**生成代理对象。需要提前生成代理的情况应该是： ab循环依赖且均需要被代理（也可以仅a需要代理），先a后b，a没有先执行代理，而是先填充b，b填充属性时，三级缓存中找到a，然后执行a的对象工厂的getObject方法获取，这个时候会代理a，然后填充给b对象，然后b对象继续初始化，生成自己的代理，完成后b移入一级缓存，并返回给a，a继续初始化，后续a就不需要重复生成代理对象了。

A，B循环依赖，先初始化A，先暴露一个半成品A，再去初始化依赖的B，初始化B时如果发现B依赖A，也就是循环依赖，就注入半成品A，之后初始化完毕B，再回到A的初始化过程时就解决了循环依赖，在这里只需要一个Map能缓存半成品A就行了，也就是二级缓存就够了，但是这个二级缓存存的是Bean对象，如果这个对象存在代理，那应该注入的是代理，而不是Bean，此时二级缓存无法及缓存Bean，又缓存代理，因此三级缓存做到了缓存 工厂 ，也就是生成代理，这我的理解：总结起来：二级缓存就能解决缓存依赖，三级缓存解决的是代理。

## IOC

你不用手动创建对象，而只需要描述创建它的方式。

利用Java的反射机制，将创建对象的方式和时机交给框架，由容器管理不同bean之间的依赖关系，实现松耦合，有利于功能复用。

DI：依赖注入，即应用程序在运行时依靠IOC容器来自动注入需要的外部依赖。

IOC注入方式：构造器注入，setter方法注入，注解注入。

## **BeanFactory和ApplicationContext**

都可以当作Spring容器，ApplicationCountext是BeanFactory的子接口。

BeanFactory：Spring里面最底层的接口，包含了各种Bean的定义，读取bean配置文档，管理bean的加载、实例化，控制bean的生命周期，维护bean之间的依赖关系。

ApplicationContext接口作为BeanFactory的派生，除了提供BeanFactory所具有的功能外，还提供了更完整的框架功能：

①继承MessageSource，因此支持国际化。

②统一的资源文件访问方式。

③提供在监听器中注册bean的事件。

④同时加载多个配置文件。

⑤载入多个（有继承关系）上下文 ，使得每一个上下文都专注于一个特定的层次，比如应用的web层。

BeanFactory延迟加载。

ApplicationContext容器创建后就创建所有bean（单例），有利于检查依赖是否注入。

## 容器中的单例bean是线程安全的吗

Spring框架并没有对单例bean进行任何多线程的封装处理。关于单例bean的线程安全和并发问题需要开发者自行去搞定。但实际上，大部分的Spring bean并没有可变的状态(比如Serview类和DAO类)，所以在某种程度上说Spring的单例bean是线程安全的。如果你的bean是有状态的话（比如 View Model 对象，有属性），就需要自行保证线程安全。最浅显的解决办法就是将多态bean的作用域由“singleton”变更为“prototype”。

## AOP

面向切面编程。用于将一些与业务无关，但在多个地方产生相同影响的部分抽取出来并封装成一个可重用的模块。避免重复编程，提高系统的可维护性。用于日志、权限、事务处理。

实现方式：

静态代理：在编译阶段就将切面织入到目标对象中产生一个完整的字节码。

动态代理：每次运行时都生成一个临时的AOP对象，这个AOP对象包含了目标对象的全部方法，并且在特定的切点做了增强处理，并回调原对象的方法。

实现：

- JDK动态代理，只提供接口的代理。核心InvocationHandler接口和Proxy类，InvocationHandler 通过invoke()方法反射来调用目标类中的代码，动态地将横切逻辑和业务编织在一起；接着，Proxy利用 InvocationHandler动态创建一个符合某一接口的的实例,  生成目标类的代理对象。

- CGLIB（Code Generation Library），是一个代码生成的类库，可以在运行时动态的生成指定类的一个子类对象，并覆盖其中特定方法并添加增强代码，从而实现AOP。CGLIB是通过继承的方式做的动态代理，因此如果某个类被标记为final，那么它是无法使用CGLIB做动态代理的。

## AOP关键词

（1）切面（Aspect）：被抽取的公共模块，可能会横切多个对象。 在Spring AOP中，切面可以使用通用类（基于模式的风格） 或者在普通类中以 @AspectJ 注解来实现。

（2）连接点（Join point）：指方法，在Spring AOP中，一个连接点 总是 代表一个方法的执行。 

（3）通知（Advice）：在切面的某个特定的连接点（Join point）上执行的动作。通知有各种类型，其中包括“around”、“before”和“after”等通知。许多AOP框架，包括Spring，都是以拦截器做通知模型， 并维护一个以连接点为中心的拦截器链。

（4）切入点（Pointcut）：切入点是指 我们要对哪些Join point进行拦截的定义。通过切入点表达式，指定拦截的方法，比如指定拦截add*、search*。

（5）引入（Introduction）：（也被称为内部类型声明（inter-type declaration））。声明额外的方法或者某个类型的字段。Spring允许引入新的接口（以及一个对应的实现）到任何被代理的对象。例如，你可以使用一个引入来使bean实现 IsModified 接口，以便简化缓存机制。

（6）目标对象（Target Object）： 被一个或者多个切面（aspect）所通知（advise）的对象。也有人把它叫做 被通知（adviced） 对象。 既然Spring AOP是通过运行时代理实现的，这个对象永远是一个 被代理（proxied） 对象。

（7）织入（Weaving）：指把增强应用到目标对象来创建新的代理对象的过程。Spring是在运行时完成织入。

切入点（pointcut）和连接点（join point）匹配的概念是AOP的关键，这使得AOP不同于其它仅仅提供拦截功能的旧技术。 切入点使得定位通知（advice）可独立于OO层次。 例如，一个提供声明式事务管理的around通知可以被应用到一组横跨多个对象中的方法上（例如服务层的所有业务操作）。

### 动态代理

1,动态代理

动态代理要求被代理对象必须实现接口。

2，CGLIB代理

第三方代理技术，原理是通过对对象的继承进行代理，即要求类不是final。

Sping的代理是两种代理方式的混合。优先使用动态代理。