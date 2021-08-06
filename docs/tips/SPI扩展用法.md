# SPI扩展用法
## 原生SPI使用
+ 定义接口和实现,一个Robot接口,2个实现
```java
public interface Robot {
    void sayHello();
}
public class OptimusPrime implements Robot {
    @Override
    public void sayHello() {
        System.out.println("Hello, I am Optimus Prime.");
    }
}
public class Bumblebee  implements  Robot {

    @Override
    public void sayHello() {
        System.out.println("Hello, I am Bumblebee.");
    }
}
```
+ 添加资源文件
在`resources`文件夹下添加`META-INF\services`定义文件

文件名为接口全路径`spi.Robot`,内容为接口全路径
```yml
#spi.Robot接口实现
spi.OptimusPrime
spi.Bumblebee
```
+ 使用
利用`ServiceLoader`的load方法进行加载
```java
public class TestJavaSpi {
    public static void main(String[] args) {
        ServiceLoader<Robot> robots = ServiceLoader.load(Robot.class);
        for (Robot robot : robots) {
            robot.sayHello();
        }
    }
}
```
有了基本的SPI知识后可以进行一些扩展使用
----- 

## 扩展SPI
基于JDK的SPI,我们可以定义一个全局的SPI处理,所有的SPI都在上面初始化,并且用map去缓存SPI实现,代码如下
```java
public abstract class ShardingServiceLoader {

    private static final Map<Class<?>, Collection<Object>> SERVICES = new ConcurrentHashMap<>();

    public static void register(final Class<?> serviceInterface) {
        if (!SERVICES.containsKey(serviceInterface)) {
            SERVICES.put(serviceInterface, load(serviceInterface));
        }
    }

    private static <T> Collection<Object> load(final Class<T> serviceInterface) {
        Collection<Object> result = new LinkedList<>();
        for (T o : ServiceLoader.load(serviceInterface)) {
            result.add(o);
        }
        return result;
    }

    public static <T> Collection<T> getServiceInstances(final Class<T> service) {
        return (Collection<T>) SERVICES.getOrDefault(service, Collections.emptyList());
    }

    public static <T> Collection<T> newServiceInstances(final Class<T> service) {
        return SERVICES.containsKey(service) ? SERVICES.get(service).stream().map(each -> (T) newServiceInstance(each.getClass())).collect(Collectors.toList()) : Collections.emptyList();
    }

    private static Object newServiceInstance(final Class<?> clazz) {
        try {
            return clazz.newInstance();
        } catch (InstantiationException | IllegalAccessException e) {
            throw new IllegalArgumentException(e);
        }
    }
}
```

### 根据特定类型,获取对应的处理SPI
> 使用场景背景: 根据传入数据源类型(oracle,mysql),初始化不同的DataSource

先看实现,定义接口
```java
public interface TypedSPI {
    //区分类型
    String getType();

    default Properties getProps() {
        return new Properties();
    }
    //传入配置
    default void setProps(final Properties props) {

    }
}

```
然后居于全局的`ShardingServiceLoader`进行扩展
```java
public abstract class TypedSPIRegistry {

    public static <T extends TypedSPI> Optional<T> findRegisteredService(final Class<T> typedSPIClass) {
        return ShardingServiceLoader.newServiceInstances(typedSPIClass).stream().findFirst();
    }

    public static <T extends TypedSPI> Optional<T> findRegisteredService(final Class<T> typedSPIClass, final String type, final Properties properties) {
        //核心逻辑,获取所有,进行筛选,然后进行配置复制传入具体实现
        Optional<T> optionalService = ShardingServiceLoader.newServiceInstances(typedSPIClass).stream().filter(each -> each.getType().equalsIgnoreCase(type)).findFirst();
        if (optionalService.isPresent()) {
            T result = optionalService.get();
            copyProperties(result, properties);
            return Optional.of(result);
        }
        return Optional.empty();
    }

    private static <T extends TypedSPI> void copyProperties(T result, Properties properties) {
        if (null != properties) {
            Properties target = new Properties();
            properties.forEach((key, value) -> target.put(key, value));
            result.setProps(target);
        }
    }

    public static <T extends TypedSPI> T getRegisteredService(final Class<T> typedSPIClass, final String type, final Properties properties) {
        Optional<T> result = findRegisteredService(typedSPIClass);
        Preconditions.checkArgument(result.isPresent(), "Not found type %s", type);
        return result.get();
    }

    public static <T extends TypedSPI> T getRegisteredService(final Class<T> typedSPIClass) {
        Optional<T> result = findRegisteredService(typedSPIClass);
        Preconditions.checkArgument(result.isPresent(), "Not found type %s", typedSPIClass.getName());
        return result.get();
    }
}
```

测试用例如下
```java
//定义SPI基于类型筛选功能
public interface TypedSPIFixture extends TypedSPI {
}
//定义mysql实现
public class MysqlHandler implements  TypedSPIFixture {
    @Override
    public String getType() {
        return "mysql";
    }
}
//定义oracle实现
public class OracleHandler implements TypedSPIFixture {
    @Override
    public String getType() {
        return "oracle";
    }
}
```
最终代码,根据传入不同类型进行获取不同的处理器
```java
    @Test
    public void testFindRegisteredServiceTypedSPIClass() throws Exception {
        TypedSPIFixture oracleHandler = TypedSPIRegistry.findRegisteredService(TypedSPIFixture.class, "oracle", null).get();
        assertThat(oracleHandler, instanceOf(OracleHandler.class));

        TypedSPIFixture mysqlHandler = TypedSPIRegistry.findRegisteredService(TypedSPIFixture.class, "mysql", null).get();
        assertThat(mysqlHandler, instanceOf(MysqlHandler.class));
    }
```




### 根据传入不同配置,获取对应处理的SPI
> 场景: 有了按类型字符串进行获取处理的,更近一步,可以更具传入的类不同,进行不同的处理,并且有顺序地进行处理.思想和上面的一样,只是传入的参数不一样

代码如下
```java
//定义带排序功能的SPI
public interface OrderedSPI<T> {

    int getOrder();

    Class<T> getTypeClass();
}
//功能实现
public abstract class OrderedSPIRegistry {


    public static <T extends OrderedSPI<?>> Map<Class<?>, T> getRegisteredServicesByClass(final Collection<Class<?>> types, final Class<T> orderedSPIClass) {
        Collection<T> registeredServices = getRegisteredServices(orderedSPIClass);
        Map<Class<?>, T> result = new LinkedHashMap<>(registeredServices.size(), 1);
        for (T each : registeredServices) {
            //核心在此,进行类型筛选
            types.stream().filter(type -> each.getTypeClass() == type).forEach(type -> result.put(type, each));
        }
        return result;
    }

    public static <T extends OrderedSPI<?>> Collection<T> getRegisteredServices(final Class<T> orderedSPIClass) {
        return getRegisteredServices(orderedSPIClass, Comparator.naturalOrder());
    }

    private static <T extends OrderedSPI<?>> Collection<T> getRegisteredServices(final Class<T> orderedSPIClass, final Comparator<Integer> comparator) {
        //进行排序
        Map<Integer, T> result = new TreeMap<>(comparator);
        for (T each : ShardingServiceLoader.getServiceInstances(orderedSPIClass)) {
            Preconditions.checkArgument(!result.containsKey(each.getOrder()), "Found same order %s with %s and %s", each.getOrder(), result.get(each.getOrder()), each);
            result.put(each.getOrder(), each);
        }
        return result.values();
    }


}
```
看下具体测试用例
```java
//定义顶层配置接口,也就是用于类型判断的父接口
public interface FixtureBaseInterface {
}
//基于父功能点扩展开的功能1
public interface FixtureFunction1Interface extends FixtureBaseInterface {
}
//基于父功能点扩展开的功能2
public interface FixtureFunction2Interface extends FixtureBaseInterface {
}
//定义SPI下所有的功能扩展接口,看到指定的筛选类型为FixtureBaseInterface
public interface OrderedSPIFixture<T extends FixtureBaseInterface> extends OrderedSPI<T> {
}
//具体功能实现
public class OrderedSPIFixtureImpl implements OrderedSPIFixture<FixtureFunction1Interface> {

    @Override
    public int getOrder() {
        return 1;
    }

    @Override
    public Class<FixtureFunction1Interface> getTypeClass() {
        return FixtureFunction1Interface.class;
    }

}
public class OrderedSPIFixtureSecondImpl implements OrderedSPIFixture<FixtureFunction2Interface> {
    @Override
    public int getOrder() {
        return 2;
    }

    @Override
    public Class<FixtureFunction2Interface> getTypeClass() {
        return FixtureFunction2Interface.class;
    }

}

```
测试用例
```java
    @Test
    public void testGetRegisteredServicesByClass() throws Exception {
        //传入的FixtureFunction2Interface类型,获取对应的处理,支持多个传入
        Map<Class<?>, OrderedSPIFixture> registeredServicesByClass = OrderedSPIRegistry.getRegisteredServicesByClass(Collections.singleton(FixtureFunction2Interface.class), OrderedSPIFixture.class);
        OrderedSPIFixture orderedSPIFixture = registeredServicesByClass.get(FixtureFunction2Interface.class);
        assertThat(orderedSPIFixture, instanceOf(OrderedSPIFixtureSecondImpl.class));
    }

```
## 总结
+ 基于JDK的SPI扩展,可以更细化点获取方式
+ 基于类型细化SPI扩展
+ 基于字符串细化SPI扩展




