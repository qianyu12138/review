# Java反射

### class对象的内容

- 成员变量 Field[] fields
- 构造方法 Constructor[] cons
- 成员方法 Method[] methods

### 获取class对象的三种方式

Java代码在计算机中经历的三个阶段：

- Source源码阶段，编译器将.java文件编译成.class字节码文件
- Class类对象阶段，类加载器将.class文件加载到虚拟机，生成类结构和class对象
- Runtime运行时阶段

对应的三种获取class对象的方式：

- Source源码阶段：Class.forName("全包名")
- Class类对象阶段：类名.class
- Runtime运行时阶段：对象.getClass()

class文件在程序运行中只会被加载一次，无论通过哪种方式获取的class对象都是同一个。

### class对象的功能

#### 获得

##### 获得成员变量

> Field[] getFields()：获取所有public修饰的成员变量
> Field getField(String name)：获取指定名称的以public修饰的成员变量

> Field[] getDeclaredFields()：获取所有成员变量，不考虑修饰符
> Field getDeclaredField(String name)

##### 获取构造方法

> Constructor<?>[] getConstructors()
>  Constructor<T> getConstructor(Class<?>... parameterTypes)

> Constructor<T> getDeclaredConstructor(Class<?>... parameterTypes)
> Constructor<?>[] getDeclaredConstructors()

##### 获取成员方法

> Method[] getMethods()
> Method getMethod(String name, Class<?>... parameterTypes)

> Method[] getDeclaredMethods()
> Method getDeclaredMethod(String name, Class<?>... parameterTypes)

#### 使用

##### Field对象的方法

由于Declared的方法会破坏封装性，Java提供了访问权限修饰符的安全检查机制，可通过

>  field.setAccessible(true);

忽略，也叫暴力反射。

常用方法：

```java
public void set(Object obj, Object value);
public Object get(Object obj);
```

##### Contructor对象的方法

常用方法

```java
public T newInstance(Object ... initargs)
```

获得空参构造方法并生成实例可以以clazz.newInstance()方式代替。

##### Method对象的方法

常用方法

```java
public Object invoke(Object obj, Object... args)
```

