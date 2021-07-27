# spring回调机制-aware接口
## 定义

由启动容器类进行分析,`AbstractApplicationContext`中的`prepareBeanFactory`方法

```java
    /**
	 * Configure the factory's standard context characteristics,
	 * such as the context's ClassLoader and post-processors.
	 * @param beanFactory the BeanFactory to configure
	 */
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);
        }
```
其中
```java
		// Configure the bean factory with context callbacks.
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
```
标注了回调方式为`ApplicationContextAwareProcessor`接口,属于`BeanPostProcessor`的一种,
初始化调用栈如下
```java
************************************************************
java.lang.Thread.getStackTrace() 1,559 <- 
org.github.hoorf.spring.aware.SpringContextUtil.printTrack() 22 <- 
org.github.hoorf.spring.aware.SpringContextUtil.setApplicationContext() 15 <- 
org.springframework.context.support.ApplicationContextAwareProcessor.invokeAwareInterfaces() 123 <- 
org.springframework.context.support.ApplicationContextAwareProcessor.postProcessBeforeInitialization() 100 <- 
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsBeforeInitialization() 416 <- 
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean() 1,788 <- 
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean() 595 <- 
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean() 517 <- 
org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0() 323 <- 
org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton() 222 <- 
org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean() 321 <- 
org.springframework.beans.factory.support.AbstractBeanFactory.getBean() 202 <- 
org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons() 882 <- 
org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization() 878 <- 
org.springframework.context.support.AbstractApplicationContext.refresh() 550 <- 
org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh() 141 <- 
org.springframework.boot.SpringApplication.refresh() 747 <- 
org.springframework.boot.SpringApplication.refreshContext() 397 <- 
org.springframework.boot.SpringApplication.run() 315 <- 
org.springframework.boot.SpringApplication.run() 1,226 <- 
org.springframework.boot.SpringApplication.run() 1,215 <- 
org.github.hoorf.spring.BootApplication.main() 9
************************************************************
```

可以看到最终的调用链上的方法`applyBeanPostProcessorsBeforeInstantiation`和`applyBeanPostProcessorsAfterInitialization`

```java
	protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
		Object bean = null;
		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
			// Make sure bean class is actually resolved at this point.
			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
				Class<?> targetType = determineTargetType(beanName, mbd);
				if (targetType != null) {
					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
					if (bean != null) {
						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
					}
				}
			}
			mbd.beforeInstantiationResolved = (bean != null);
		}
		return bean;
	}
```

最终可以看到其实现机制是实现了`BeanPostProcessor`,进行调用,再看下上文提到aware所实现`ApplicationContextAwareProcessor`

```java
    @Override
	@Nullable
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
				bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
				bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)){
			return bean;
		}

		AccessControlContext acc = null;

		if (System.getSecurityManager() != null) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}

		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
			invokeAwareInterfaces(bean);
		}

		return bean;
	}

    private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof EnvironmentAware) {
			((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
		}
		if (bean instanceof EmbeddedValueResolverAware) {
			((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
		}
		if (bean instanceof ResourceLoaderAware) {
			((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
		}
		if (bean instanceof ApplicationEventPublisherAware) {
			((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
		}
		if (bean instanceof MessageSourceAware) {
			((MessageSourceAware) bean).setMessageSource(this.applicationContext);
		}
		if (bean instanceof ApplicationContextAware) {
			((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
		}
	}

```

可以看到回调的aware种类
+ EnvironmentAware
+ EmbeddedValueResolverAware
+ ResourceLoaderAware
+ ApplicationEventPublisherAware
+ MessageSourceAware
+ ApplicationContextAware



## 总结
aware实现的机制也是`BeanPostProcessor`
```java
public interface BeanPostProcessor {


	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

}
```
主要实现方法`postProcessBeforeInitialization`,在bean初始化之前执行,然后对相关实现接口进行回调


