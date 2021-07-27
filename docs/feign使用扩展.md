# feign使用扩展
## 背景
feign在使用过程中,一般先定义接口,再定义具体实现的controller,如下所示


```java
@FeignClient(name = "shore-app")
public interface StoreApi {

    @GetMapping("/api/store/getStoreNum")
    Integer getStoreNum();

    @GetMapping("/api/store/getStoreNumByCondition1")
    Integer getStoreNumByCondition1();


    @GetMapping("/api/store/getStoreNumByCondition2")
    Integer getStoreNumByCondition2();
}
```
实现定义

```java
@RestController
public class StoreController implements StoreApi {


    @Override
    public Integer getStoreNum() {
        return Integer.MAX_VALUE;
    }

    @Override
    public Integer getStoreNumByCondition1() {
        return Integer.MAX_VALUE;
    }

    @Override
    public Integer getStoreNumByCondition2() {
        return Integer.MAX_VALUE;
    }
}
```
这种方式去提供Feign给其他模块调用也是比较简洁的,不过在定义`controller`很多人做法是把url公共的部分抽取出来,如下
```java
@FeignClient(name = "shore-app")
@RequestMapping("/api/store")
public interface StoreApi {

    @GetMapping("/getStoreNum")
    Integer getStoreNum();

    @GetMapping("/getStoreNumByCondition1")
    Integer getStoreNumByCondition1();


    @GetMapping("/getStoreNumByCondition2")
    Integer getStoreNumByCondition2();
}
```
然后会发现项目启动报错,有相同的url路径,启动不起来,报错如下

```java
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'requestMappingHandlerMapping' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/WebMvcAutoConfiguration$EnableWebMvcConfiguration.class]: Invocation of init method failed; nested exception is java.lang.IllegalStateException: Ambiguous mapping. Cannot map 'org.github.hoorf.store.api.StoreApi' method 
org.github.hoorf.store.api.StoreApi#getStoreNumByCondition2()
to {GET /api/store/getStoreNumByCondition2}: There is already 'storeController' bean method
org.github.hoorf.store.controller.StoreController#getStoreNumByCondition2() mapped.
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1796) ~[spring-beans-5.2.5.RELEASE.jar:5.2.5.RELEASE]

```

报错原因是`@RequestMapping`标注上,被spring识别为controller

---

本文介绍的是一种新的feign定义模式去解决这问题,既能保持原有的`controller`使用习惯,又能将feign定义的接口复用到其他项目

## 实现
工程目录分包依赖情况
```
e1
|--e1-api
|     |--e1-store-api
|     |--e1-account-api
|     |--e1-common
|--e1-acount
|--e1-store     
```
其中`e1-account`,`e1-store`分别依赖`e1-api`,`e1-api`为所有api的聚合依赖

1. 自定义Feign注解
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Inherited
@FeignClient
@Conditional(AdaptiveFeignCondition.class)
public @interface AdaptiveFeign {

    @AliasFor(annotation = FeignClient.class)
    String value() default "";

    @AliasFor(annotation = FeignClient.class)
    Class<?> fallback() default void.class;

    @AliasFor(annotation = FeignClient.class)
    Class<?> fallbackFactory() default void.class;
}
```
2. 定义注解启用条件

基本逻辑就是,如果识别到实现类有controller则不启用Feign注解,识别不到则启用Feign注解
```java
@Slf4j
public class AdaptiveFeignCondition extends SpringBootCondition {
    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
        ClassMetadata classMetadata = (ClassMetadata) metadata;

        String className = classMetadata.getClassName();
        Class interfaceClass = null;
        try {
            interfaceClass = Class.forName(className);
        } catch (ClassNotFoundException e) {
        }

        Set<Class<?>> classSet = ReflectionUtils.getSubTypesOf(interfaceClass);

        for (Class<?> implClass : classSet) {
            Controller annotation = AnnotationUtils.findAnnotation(implClass, Controller.class);
            if (null != annotation) {
               // log.debug("{} exits Controller.class ,no init feign {}", implClass.getName(), interfaceClass.getName());
                return ConditionOutcome.noMatch(new StringBuffer(implClass.getName()).append(" exits implement").toString());
            }
        }
        return ConditionOutcome.match();
    }
}
```

注意版本使用0.9.12存在bug,0.9.11正常使用
```xml
<dependency>
    <groupId>org.reflections</groupId>
    <artifactId>reflections</artifactId>
    <version>0.9.11</version>
</dependency>
```

## 效果

在 `account`模块中使用
```java
    @Autowired
    private AccountApi accountApi;
```
注入的是Spring中的`AccountApi`具体实现,也就是controller,走的是内部调用

在 `store`模块中使用
注入的是`AccountApi`Feign的实现,走的是远程调用

具体工程代码[https://github.com/hoorf/arch/tree/master/arch-example-1](https://github.com/hoorf/arch/tree/master/arch-example-1)