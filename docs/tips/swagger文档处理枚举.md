# swagger文档处理枚举

## 背景
处理接口返回时,经常有状态值返回,如果状态值变更,所用到的地方信息都要改,本文介绍的是一种随着后台枚举类变化,返回的注释也同时变化的处理方法

## 实现
根据swagger提供的扩展方式`ModelPropertyBuilderPlugin`进行扩展

### 自定义swagger文档注解
```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ApiModelPropertyEnum {

    /**
     * 枚举类对象
     *
     * @return
     */
    Class<? extends BaseEnum> value();

    String example() default "";

    /**
     * 是否隐藏
     *
     * @return
     */
    boolean hidden() default false;

    /**
     * 是否必须
     *
     * @return
     */
    boolean required() default true;

    String dataType() default "";

    String enumDesc() default "";

}
```

### 定义插件处理逻辑

```java
public class SmartSwaggerApiModelEnumPlugin implements ModelPropertyBuilderPlugin {

    @Override
    public void apply(ModelPropertyContext context) {
        Optional<ApiModelPropertyEnum> enumOptional = Optional.absent();

        if (context.getAnnotatedElement().isPresent()) {
            enumOptional = enumOptional.or(findApiModePropertyAnnotation(context.getAnnotatedElement().get()));
        }
        if (context.getBeanPropertyDefinition().isPresent()) {
            enumOptional = enumOptional.or(findPropertyAnnotation(context.getBeanPropertyDefinition().get(), ApiModelPropertyEnum.class));
        }

        if (enumOptional.isPresent()) {
            ApiModelPropertyEnum anEnum = enumOptional.get();
            //获取枚举所有信息
            String enumInfo = BaseEnum.getInfo(anEnum.value());
            context.getBuilder()
                    .required(enumOptional.transform(toIsRequired()).or(false))
                    .description(anEnum.enumDesc() + ":" + enumInfo)
                    .example(enumOptional.transform(toExample()).orNull())
                    .isHidden(anEnum.hidden());
        }
    }

    @Override
    public boolean supports(DocumentationType delimiter) {
        return SwaggerPluginSupport.pluginDoesApply(delimiter);
    }

    static Function<ApiModelPropertyEnum, Boolean> toIsRequired() {
        return annotation -> annotation.required();
    }

    public static Optional<ApiModelPropertyEnum> findApiModePropertyAnnotation(AnnotatedElement annotated) {
        return Optional.fromNullable(AnnotationUtils.getAnnotation(annotated, ApiModelPropertyEnum.class));
    }

    static Function<ApiModelPropertyEnum, String> toExample() {
        return annotation -> {
            String example = annotation.example();
            if (StringUtils.isBlank(example)) {
                return "";
            }
            return example;
        };
    }
}
```

### 定义到Bean

```java
    @Bean
    @Order(SwaggerPluginSupport.SWAGGER_PLUGIN_ORDER + 1)
    //定义到最后处理
    public SmartSwaggerApiModelEnumPlugin swaggerEnum(){
        return new SmartSwaggerApiModelEnumPlugin();
    }
```


## 效果

![文档图](/img/swagger文档处理枚举_1.png)