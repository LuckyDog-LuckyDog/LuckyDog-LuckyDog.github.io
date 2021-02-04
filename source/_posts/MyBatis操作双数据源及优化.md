---
title: MyBatis操作双数据源及优化
date: 2021-02-04 14:40:33
tags: 
- MyBatis
- Druid
- MYSQL
categories:
- Java
- spring boot
---

本文记录用MyBatis对mysql数据库的不同数据源进行操作,可实现类似于之前JPA的动态切换, 其中利用阿里的druid对双数据源进行监控
<!--more-->


# 1 MyBatis

## 1.1 配置文件
```yml
spring:
  aop:
    proxy-target-class: true
    auto: true
  datasource:
    druid:
      db1:
        url:  jdbc:mysql://192.168.5.27:3306/A?autoReconnect=true&useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=CONVERT_TO_NULL&useSSL=false&serverTimezone=CTT
        username: root
        password: 123456
        driver-class-name: com.mysql.cj.jdbc.Driver
        type: com.alibaba.druid.pool.DruidDataSource
        initialSize: 5
        minIdle: 5
        maxActive: 20
        maxWait: 60000
        timeBetweenEvictionRunsMillis: 60000
        minEvictableIdleTimeMillis: 300000
        validationQuery: SELECT 1 FROM DUAL
        testWhileIdle: true
        testOnBorrow: false
        testOnReturn: false
      db2:
        url:  jdbc:mysql://192.168.5.27:3306/B?autoReconnect=true&useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=CONVERT_TO_NULL&useSSL=false&serverTimezone=CTT
        username: root
        password: 123456
        driver-class-name: com.mysql.cj.jdbc.Driver
        type: com.alibaba.druid.pool.DruidDataSource
        initialSize: 5
        minIdle: 5
        maxActive: 20
        maxWait: 60000
        timeBetweenEvictionRunsMillis: 60000
        minEvictableIdleTimeMillis: 300000
        validationQuery: SELECT 1 FROM DUAL
        testWhileIdle: true
        testOnBorrow: false
        testOnReturn: false
```

## 1.2 数据源配置

```java

/**
 * Description: aop的实现的数据源切换aop切点，实现mapper类寻找，找到所属大本营以后，如db1Aspect(),则会在调用db1()前面之前，进行数据源的切换。
 * @author:xiaoLiu
 */

@Component
@Order(value = -100) //值越小,越早执行
@Aspect
public class DataSourceAspect {
    private Log log = Log.get();

    @Pointcut("execution(* cn.xiaoliu.business.siteBill.mapper.db1.*.*(..))")
    private void db1Aspect() {
    }

    @Pointcut("execution(* cn.xiaoliu.business.siteBill.mapper.db2.*.*(..))")
    private void db2Aspect() {
    }

    @Before("db1Aspect()")
    public void db1() {
        log.info(">>>>>>>>目前使用db1数据源...");
        CurrentDataSourceContext.setDataSourceType(DBTypeEnum.db1.getValue()); }

    @Before("db2Aspect()")
    public void db2() {
        log.info(">>>>>>>目前使用db2数据源...");
        CurrentDataSourceContext.setDataSourceType(DBTypeEnum.db2.getValue());
    }
```
数据源枚举
```java
public enum DBTypeEnum {
    /**
     * 数据源名称
     */
    db1("db1"), db2("db2");

    private String value;

    DBTypeEnum(String value) {
        this.value = value;
    }

    public String getValue() {
        return value;
    }
}
```

## 1.3 设置指定数据源(动态)

```java
public class DynamicDataSource extends AbstractRoutingDataSource {
    private static final Log LOGGER = Log.get();

    @Override
    protected Object determineCurrentLookupKey() {
        String datasource = CurrentDataSourceContext.getDataSourceType();
        LOGGER.debug(">>>>使用数据源 {}", datasource);
        return datasource;
    }
}
```

## 1.4 MyBatis配置类

```java

@Configuration
@MapperScan(basePackages = {"cn.mallwash.**.mapper"})
public class MybatisConfig {
    private final Log log = Log.get();
/**
     * 自定义公共字段自动注入
     *这里不用注入,因为多源,自动注入没啥软用 , 需要手动注入
     */
//    @Bean
    public MetaObjectHandler metaObjectHandler() {
        return new CustomMetaObjectHandler();
    }
    
    @Bean
    public PaginationInterceptor paginationInterceptor() {
        return new PaginationInterceptor();
    }
   //======================多数据源的配置======================================

    @Bean(name = "db1") //数据源名称
    @ConfigurationProperties(prefix = "spring.datasource.druid.db1")
    public DruidDataSource db1() {
        log.info(">>>>>>开始配置数据源1");
        return DruidDataSourceBuilder.create().build();
    }

    @Bean(name = "db2")
    @ConfigurationProperties(prefix = "spring.datasource.druid.db2")
    public DruidDataSource db2() {
        log.info(">>>>>>开始配置数据源2");
        return DruidDataSourceBuilder.create().build();
    }

    /**
     * 动态数据源配置
     *
     * @return
     */
    @Bean
    @Primary
    public DataSource multipleDataSource(@Qualifier("db1") DataSource db1,
                                         @Qualifier("db2") DataSource db2) {
        //创建一个动态数据源装入A和B两者的源
        DynamicDataSource dynamicDataSource = new DynamicDataSource();
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put(DBTypeEnum.db1.getValue(), db1);
        targetDataSources.put(DBTypeEnum.db2.getValue(), db2);
        dynamicDataSource.setTargetDataSources(targetDataSources);
        // 程序默认数据源，这个要根据程序调用数据源频次，经常把常调用的数据源作为默认
        dynamicDataSource.setDefaultTargetDataSource(db1);
        return dynamicDataSource;
    }


    @Bean("sqlSessionFactory")
    public SqlSessionFactory sqlSessionFactory(ApplicationContext applicationContext) throws Exception {
        MybatisSqlSessionFactoryBean sqlSessionFactory = new MybatisSqlSessionFactoryBean();
        sqlSessionFactory.setDataSource(multipleDataSource(db1(), db2()));

        MybatisConfiguration configuration = new MybatisConfiguration();
        //多源下, 配置文件会失效,需要手动进行配置
        GlobalConfig globalConfig=new GlobalConfig();
        //自动注入自定义数据
        globalConfig.setMetaObjectHandler(metaObjectHandler());
        globalConfig.setBanner(false);
        configuration.setJdbcTypeForNull(JdbcType.NULL);
        configuration.setMapUnderscoreToCamelCase(true);
        configuration.setCacheEnabled(true);
        configuration.setLazyLoadingEnabled(true);
        configuration.setLogImpl(org.apache.ibatis.logging.nologging.NoLoggingImpl.class);
        configuration.setMultipleResultSetsEnabled(true);
        sqlSessionFactory.setConfiguration(configuration);
        sqlSessionFactory.setGlobalConfig(globalConfig);
        //这是为了兼容之前的代码位置 , 注意这里的位置必须设置为classpath*:前缀的
        String MAPPER_LOCATION1 = "classpath*:cn/xiaoliu/**/mapping/*.xml";
        //A库位置
        String MAPPER_LOCATION2 = "classpath*:cn/xiaoliu/**/mapping/db1/*.xml";
        //B库位置
        String MAPPER_LOCATION3 = "classpath*:cn/xiaoliu/**/mapping/db2/*.xml";
		//设置xml资源的位置
        sqlSessionFactory.setMapperLocations(
                resolveMapperLocations(MAPPER_LOCATION1,MAPPER_LOCATION2,MAPPER_LOCATION3)
        );
        //PerformanceInterceptor(),OptimisticLockerInterceptor()
        //添加分页功能
        sqlSessionFactory.setPlugins(paginationInterceptor());
//        sqlSessionFactory.setGlobalConfig(globalConfiguration()); //注释掉全局配置，因为在xml中读取就是全局配置
        return sqlSessionFactory.getObject();
    }


	/**
		xml资源解析器
	**/
    private Resource[] resolveMapperLocations(String ... paths) {
        ResourcePatternResolver resourceResolver = new PathMatchingResourcePatternResolver();
        List<String> mapperLocations = new ArrayList<>(Arrays.asList(paths));
        List<Resource> resources = new ArrayList<>();
        if (ObjectUtil.isNotEmpty(mapperLocations)) {
            for (String mapperLocation : mapperLocations) {
                try {
                    Resource[] mappers = resourceResolver.getResources(mapperLocation);
                    resources.addAll(Arrays.asList(mappers));
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        int size = resources.size();
        return resources.toArray(new Resource[size]);
    }
//======================德鲁伊监控多数据源的配置======================================
    /**
     * druid配置
     */
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.druid.db1")
    public DruidProperties druidProperties() {
        return new DruidProperties();
    }
    /**
     * druid配置
     */
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.druid.db2")
    public DruidProperties druidProperties2() {
        return new DruidProperties();
    }

    /**
     * druid数据库连接池
     */
    @Bean(initMethod = "init")
    public DruidDataSource dataSource() {
//        DruidDataSource dataSource = new DruidDataSource();
        DruidDataSource dataSource = db1();
        druidProperties().config(dataSource);
        return dataSource;
    }
    /**
     * druid数据库连接池2
     */
    @Bean(initMethod = "init")
    public DruidDataSource dataSource2() {
        DruidDataSource dataSource = db2();
        druidProperties2().config(dataSource);
        return dataSource;
    }
}

```
## 1.6 德鲁伊监控配置
```java
@Configuration
public class DataSourceConfig {

    /**
     * druid监控，配置StatViewServlet
     */
    @Bean
    public ServletRegistrationBean<StatViewServlet> druidServletRegistration() {

        // 设置servlet的参数
        HashMap<String, String> statViewServletParams = CollectionUtil.newHashMap();
        statViewServletParams.put("resetEnable", "true");
        statViewServletParams.put("loginUsername", ConstantContextHolder.getDruidMonitorUsername());
        statViewServletParams.put("loginPassword", ConstantContextHolder.getDruidMonitorPassword());


        ServletRegistrationBean<StatViewServlet> registration = new ServletRegistrationBean<>(new StatViewServlet());
        registration.addUrlMappings("/druid/*");
        registration.setInitParameters(statViewServletParams);
        return registration;
    }

```