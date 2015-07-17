# Using `@Cacheable` #

Declaring cache semantics directly in the Java source code puts the declarations much closer to the affected code. There is not much danger of undue coupling, because code that is meant to have its results cached is almost always deployed that way anyway.

The ease-of-use afforded by the use of the `@Cacheable` annotation is best illustrated with an example, which is explained in the text that follows. Consider the following interface and class definition:
```
public interface WeatherDao {
    
    public Weather getWeather(String zipCode);
    
    public List<Location> findLocations(String locationSearch);
}

public class DefaultWeatherDao implements WeatherDao {
    
    @Cacheable(cacheName="weatherCache")
    public Weather getWeather(String zipCode) {
        //Some Code
    }
    
    @Cacheable(cacheName="locationSearchCache")
    public List<Location> findLocations(String locationSearch) {
        //Some Code
    }
}
```

When the above POJO is defined as a bean in a Spring IoC container, the bean instance can be made 'cacheable' by adding merely one line of XML configuration:
```
<!-- from the file 'context.xml' -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
     xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
     xmlns:ehcache="http://ehcache-spring-annotations.googlecode.com/svn/schema/ehcache-spring"
     xsi:schemaLocation="
     http://www.springframework.org/schema/beans 
     http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
     http://ehcache-spring-annotations.googlecode.com/svn/schema/ehcache-spring
     http://ehcache-spring-annotations.googlecode.com/svn/schema/ehcache-spring/ehcache-spring-1.1.xsd">

    <ehcache:annotation-driven cache-manager="ehCacheManager" />
    
    <bean id="ehCacheManager" class="org.springframework.cache.ehcache.EhCacheManagerFactoryBean"/>
    
    <bean id="weatherDao" class="x.y.service.DefaultWeatherDao"/>
</beans>
```

Also the following ehcache.xml configuration file must be present on the classpath to declare and configure the two caches.
```
<?xml version="1.0" encoding="UTF-8"?>
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd" updateCheck="false">
    <!--
     | Please see http://ehcache.sourceforge.net/documentation/configuration.html for
     | detailed information on how to configurigure caches in this file
     +-->
    <!-- Location of persistent caches on disk -->
    <diskStore path="java.io.tmpdir/EhCacheSpringAnnotationsExampleApp" />

    <defaultCache eternal="false" maxElementsInMemory="1000"
        overflowToDisk="false" diskPersistent="false" timeToIdleSeconds="0"
        timeToLiveSeconds="600" memoryStoreEvictionPolicy="LRU"/>

    <cache name="weatherCache" eternal="false"
        maxElementsInMemory="100" overflowToDisk="false" diskPersistent="false"
        timeToIdleSeconds="0" timeToLiveSeconds="300"
        memoryStoreEvictionPolicy="LRU" />

    <cache name="locationSearchCache" eternal="false"
        maxElementsInMemory="100" overflowToDisk="false" diskPersistent="false"
        timeToIdleSeconds="0" timeToLiveSeconds="300"
        memoryStoreEvictionPolicy="LRU" />
</ehcache>
```

### CacheManager Reference ###
You can omit the `cache-manager` attribute in the `<ehcache:annotation-driven/>` tag if the bean name of the `CacheManager` that you want to wire in has the name `cacheManager`. If the `CacheManager` bean that you want to dependency-inject has any other name, then you have to use the `cache-manager` attribute explicitly, as in the preceding example.

### EhCache Configuration ###
You must configure EhCache correctly to use the `@Cacheable` annotation. If you are not familiar with configuring EhCache please read their [configuration guide](http://ehcache.org/documentation/configuration.html)

### Annotation Placement ###
You can place the `@Cacheable` annotation on a method on an interface, or a public method on a class. However, the mere presence of the `@Cacheable` annotation is not enough to activate the caching behavior. The `@Cacheable` annotation is simply metadata that can be consumed by some runtime infrastructure that is `@Cacheable`-aware and that can use the metadata to configure the appropriate beans with caching behavior. In the preceding example, the `<ehcache:annotation-driven/>` element switches on the caching behavior.

Spring recommends that you only annotate methods of concrete classes with the `@Cacheable` annotation, as opposed to annotating methods of interfaces. You certainly can place the `@Cacheable` annotation on an interface (or an interface method), but this works only as you would expect it to if you are using interface-based proxies. The fact that Java annotations are not inherited from interfaces  means that if you are using class-based proxies (proxy-target-class="true") then the transaction settings are not recognized by the proxying and weaving infrastructure, and the object will not be wrapped in a caching proxy, which would be decidedly bad.

### Method visibility and `@Cacheable` ###
When using proxies, you should apply the `@Cacheable` annotation only to methods with public visibility. If you do annotate protected, private or package-visible methods with the `@Cacheable` annotation, no error is raised, but the annotated method does not exhibit the configured cachable settings.

### Self-Invocation ###
Only external method calls coming in through the proxy are intercepted. This means that self-invocation, in effect, a method within the target object calling another method of the target object, will not lead to an actual cache interception at runtime even if the invoked method is marked with `@Cacheable`.

### Table 1. `<ehcache:annotation-driven/>`  settings ###
| **Attribute** | **Default** | **Description** |
|:--------------|:------------|:----------------|
| `cache-manager` | cacheManager | The bean name of the CacheManager that is to be used to drive caching. This attribute is not required, and only needs to be specified explicitly if the bean name of the desired CacheManager is not 'cacheManager'. |
| `create-missing-caches` | false       | Should cache names from `@Cacheable` annotations that don't exist in the CacheManager be created based on the default cache or should an exception be thrown?|
| `default-cache-key-generator` | Not Set     | Default CacheKeyGenerator implementation to use. If not specified HashCodeCacheKeyGenerator will be used as the default. |
| `self-populating-cache-scope` | shared      | Are the SelfPopulatingCache wrappers scoped to the method or are they shared among all methods using each self populating cache. |
| `proxy-target-class` | false       | Applies to proxy mode only. Controls what type of caching proxies are created for classes annotated with the `@Cacheable` annotation. If the proxy-target-class attribute is set to true, then class-based proxies are created. If proxy-target-class is false or if the attribute is omitted, then standard JDK interface-based proxies are created. (See [Spring Reference Section 7.6, “Proxying mechanisms”](http://static.springsource.org/spring/docs/3.0.x/spring-framework-reference/html/aop.html#aop-proxying) for a detailed examination of the different proxy types.)|
| `order`       | Ordered.LOWEST\_PRECEDENCE | Defines the order of the caching advice that is applied to beans annotated with `@Cacheable`. (For more information about the rules related to ordering of AOP advice, see [Spring Reference Section 7.2.4.7, “Advice ordering”](http://static.springsource.org/spring/docs/3.0.x/spring-framework-reference/html/aop.html#aop-ataspectj-advice-ordering).) No specified ordering means that the AOP subsystem determines the order of the advice. |

### `@Cacheable` Settings ###
The `@Cacheable` annotation is metadata that specifies that a method must have cache semantics; for example, “cache the result of this method call in a cache named `weatherCache`, blocking other threads that are trying to make the same call with the same arguments to share the result”. The default `@Cacheable` settings are as follows:

**Cache Name is unset and required by the developer** Self Populating is `false`
**keyGeneratorName is empty, the default key generator is used.** exceptionCacheName is empty, no exception caching is done.

These default settings can be changed; the various properties of the `@Cacheable` annotation are summarized in the following table:
| **Property** | **Type** | **Description** |
|:-------------|:---------|:----------------|
| cacheName    | String   | Required name of the cache to use |
| selfPopulating | boolean  | Optional if the method should have self-populating semantics. This results in calls to the annotated method to be made within a EhCache CacheEntryFactory. This is useful for expensive shared resources that should be retrieved a minimal number of times. |
| keyGenerator | [@KeyGenerator](UsingKeyGenerator.md) | Optional inline configuration of a CacheKeyGenerator to use for generating cache keys for this method. |
| keyGeneratorName | String   | Optional bean name of a CacheKeyGenerator to use for generating cache keys for this method. |
| exceptionCacheName | String   | Optional name of the cache to use for caching exceptions. If not specified exceptions result in nothing being cached. If specified the thrown exception is stored in the cache for the generated key and re-thrown for subsequent method calls as long as it remains in the exception cache. |