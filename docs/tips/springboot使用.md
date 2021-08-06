# springboot使用

## Condition 判断注解是否启用

### 说明

首先看`Conditional`的注释

```java
Indicates that a component is only eligible for registration when all specified conditions match.
A condition is any state that can be determined programmatically before the bean definition is due to be registered (see Condition for details).
The @Conditional annotation may be used in any of the following ways:
as a type-level annotation on any class directly or indirectly annotated with @Component, including @Configuration classes
as a meta-annotation, for the purpose of composing custom stereotype annotations
as a method-level annotation on any @Bean method
If a @Configuration class is marked with @Conditional, all of the @Bean methods, @Import annotations, and @ComponentScan annotations associated with that class will be subject to the conditions.
NOTE: Inheritance of @Conditional annotations is not supported; any conditions from superclasses or from overridden methods will not be considered. In order to enforce these semantics, @Conditional itself is not declared as @Inherited; furthermore, any custom composed annotation that is meta-annotated with @Conditional must not be declared as @Inherited
```
1. 作用于类级别
2. 作用于元数据级别(注解类型的)
3. 作用于方法级别
4. 不支持注解传递

作用于类和方法比较常见,如下
```java
@ConditionalOnExpression(FeignAspectAutoConfiguration.FEIGN_ASPECT_CONDITION)
public class FeignAspectAutoConfiguration {
    @Bean
    @ConditionalOnProperty(name = FeignAspectAutoConfiguration.FEIGN_LOG_CONDITION_PROPERTY, havingValue = "true")
    public AspectJExpressionPointcut feignClientPointcut(){}
}
```

本文介绍一种用于是否启用注解的方式

首先定义注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Component
@Conditional(AdaptiveFeignCondition.class)
public @interface AdaptiveFeign {
}
```

定义conditional类
```java
public class AdaptiveFeignCondition extends SpringBootCondition {
    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return ConditionOutcome.match();
    }


}
```
主要依赖于`ConditionOutcome`的返回去设置是否启用,`ConditionOutcome.match()`表示启用,`ConditionOutcome.noMatch("no match")`表示不启用


### 总结
这种方式可用于编写自适应的启用注解,检查是否存在某些类,进行启用注解,可用以下场景
+ 动态启用feign
