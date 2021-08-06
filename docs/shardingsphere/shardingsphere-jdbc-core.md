# shardingsphere-jdbc-core 源码


## 创建数据源
核心代码
```java
    public ShardingSphereDataSource(final Map<String, DataSource> dataSourceMap, final Collection<RuleConfiguration> configurations, final Properties props) throws SQLException {
        DistMetaDataPersistRepository repository = DistMetaDataPersistRepositoryFactory.newInstance(configurations);
        metaDataContexts = new MetaDataContextsBuilder(Collections.singletonMap(DefaultSchema.LOGIC_NAME, dataSourceMap),
                Collections.singletonMap(DefaultSchema.LOGIC_NAME, configurations), props).build(new DistMetaDataPersistService(repository));
        String xaTransactionMangerType = metaDataContexts.getProps().getValue(ConfigurationPropertyKey.XA_TRANSACTION_MANAGER_TYPE);
        transactionContexts = createTransactionContexts(metaDataContexts.getDefaultMetaData().getResource().getDatabaseType(), dataSourceMap, xaTransactionMangerType);
    }
```
1. 转换配置成DistMetaDataPersistRepository
`DistMetaDataPersistRepository` 默认实现方式 `LocalDistMetaDataPersistRepository`,主要用于配置持久化,或分布式配置
同时也会初始化一个`DistMetaDataPersistService` ,用于细分各种操作(类似dao和service)
2. 初始化元数据上下文
`MetaDataContexts` 默认实现方式 `StandardMetaDataContexts`
```java
    private final DistMetaDataPersistService distMetaDataPersistService;

    private final Map<String, ShardingSphereMetaData> metaDataMap;

    private final ShardingSphereRuleMetaData globalRuleMetaData;

    private final ExecutorEngine executorEngine;
    //利用calcite来初始化sql解析器,后替换了实现
    private final OptimizeContextFactory optimizeContextFactory;

    private final ConfigurationProperties props;

    private final StateContext stateContext;

    public StandardMetaDataContexts(final DistMetaDataPersistService persistService) {
        this(persistService, new LinkedHashMap<>(), new ShardingSphereRuleMetaData(Collections.emptyList(), Collections.emptyList()),
                null, new ConfigurationProperties(new Properties()), new OptimizeContextFactory(new HashMap<>()));
    }

    public StandardMetaDataContexts(final DistMetaDataPersistService persistService, final Map<String, ShardingSphereMetaData> metaDataMap, final ShardingSphereRuleMetaData globalRuleMetaData,
                                    final ExecutorEngine executorEngine, final ConfigurationProperties props, final OptimizeContextFactory optimizeContextFactory) {
        this.distMetaDataPersistService = persistService;
        this.metaDataMap = new LinkedHashMap<>(metaDataMap);
        this.globalRuleMetaData = globalRuleMetaData;
        this.executorEngine = executorEngine;
        this.optimizeContextFactory = optimizeContextFactory;
        this.props = props;
        stateContext = new StateContext();
    }
```
`StandardMetaDataContext`构建涉及到各种配置的加载,加载方式如下
> ShardingSphereRulesBuilder 定义配置分类



+ DistributedSchemaRuleBuilder 分布式规则(读写分离之类的)
+ EnhancedSchemaRuleBuilder 增强形规则(加密,分片之类的)
+ GlobalRuleBuilder (全局)
+ DefaultKernelRuleConfigurationBuilder (内核配置)


### 加载配置
转换配置成 DistMetaDataPersistRepository
```java

```
