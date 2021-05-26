# SpringBoot自动配置原理

一切源于`@SpringBootApplication`

```java
@SpringBootApplication
public class WebDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(WebDemoApplication.class, args);
    }

}
```

它是一个派生注解：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration//派生@Configuration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
```

重要的注解有三个：

- @Configuration
- @ComponentScan
- @EnableAutoConfiguration

### Configuration

spring重要注解，所有由该注解标注的都可以视为IOC容器

```java
@Configuration
public class RedisCacheConfig {
    @Bean(name="redisTemplate")
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig();
        RedisCacheManager cacheManager = RedisCacheManager.builder(factory)
                .cacheDefaults(config)
                .build();
        return cacheManager;
    }
}
```

作为XML的替代品，作为bean定义的来源。

### ComponentScan

将所有包下的组件，而默认的basePackages就是标示类所在的路径。

### **EnableAutoConfiguration**

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
```

引入了一个Selector`AutoConfigurationImportSelector`，它的作用是借助于Spring框架原有的一个工具类`SpringFactoriesLoader`来将META-INF/spring.factories配置文件里以`org.springframework.boot.autoconfigure.EnableAutoConfiguration`为key的所有组件注册到容器中，再由这些configuration自定义需要注册到容器中的bean。

比如我写的一个start-demo中spring.factories

```factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.example.aspectlogspringbootstarter.AspectLogAutoConfiguration
```



https://www.cnblogs.com/jstarseven/p/11087157.html

