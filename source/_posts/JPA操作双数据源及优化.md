---
title: JPA操作双数据源及优化
date: 2021-02-04 10:20:34
tags: 
- JPA
- MYSQL
categories:
- Java
- spring boot
---


本文记录用JPA对数据库不同库进行配置 , 进而对不同库进行CRUB , 其中对JPA批量性操作进行优化 , 对不同数据源下事务和多线程操作
<!--more-->

# 1.JPA
## 1.1 数据源配置
```java
@Configuration //配置类
public class DataSourceConfig {
    @Primary \\优先加载该bean
    @Bean(name = "primaryDataSource") //产生一个由Spring容器管理的bean
    @Qualifier("primaryDataSource") //命名指定
    @ConfigurationProperties(prefix = "spring.datasource.primary") //配置文件加载属性
    public DataSource primaryDataSource() {
        return new HikariDataSource();
    }

    @Bean(name = "secondaryDataSource")
    @Qualifier("secondaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.secondary")
    public DataSource secondaryDatasource() {
        return new HikariDataSource();
    }

    @Bean
    public HibernateProperties hibernateProperties() {
        return new HibernateProperties();
    }
}
```

##1.2 文件配置
```yml
spring:
  profiles:
    active: dev
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false
  data:
    redis:
      repositories:
        enabled: false
server:
  port: 8089


spring:
  datasource:
    primary:
      jdbc-url: jdbc:mysql://192.168.5.8/A?createDatabaseIfNotExist=true&useUnicode=true&characterEncoding=utf-8&zeroDateTimeBehavior=convertToNull&transformedBitIsBoolean=true&autoReconnect=true&failOverReadOnly=false&serverTimezone=Asia/Shanghai
      username: root
      password: 123456
      driver-class-name: com.mysql.cj.jdbc.Driver
      minimum-idle: 10
      maximum-pool-size: 50
    secondary:
      jdbc-url: jdbc:mysql://192.168.5.8/B?createDatabaseIfNotExist=true&useUnicode=true&characterEncoding=utf-8&zeroDateTimeBehavior=convertToNull&transformedBitIsBoolean=true&autoReconnect=true&failOverReadOnly=false&serverTimezone=Asia/Shanghai
      username: root
      password: 123456
      driver-class-name: com.mysql.cj.jdbc.Driver
      minimum-idle: 10
      maximum-pool-size: 50

```
## 1.3 数据源A配置
```java
@Configuration //配置类
@EnableTransactionManagement //开启事务管理
//开启jpa属性,默认情况下，将扫描带注释的配置类的包中的存储库。
@EnableJpaRepositories( 
        entityManagerFactoryRef = "primaryEntityManagerFactory",
        transactionManagerRef = "primaryTransactionManager", //引入
        basePackages = "cn.xiaoliu.web.repository.primary") //设置仓库类
public class PrimaryConfig {
     /**
 自动加载hibernate属性spring.jpa.hibernate
   **/
    @Autowired 
    private HibernateProperties hibernateProperties;
    
   /**
   自动注入A数据源
   **/
    @Resource 
    @Qualifier("primaryDataSource")
    private DataSource primaryDataSource;

    /**
   优先A事务管理器
   **/
    @Primary 
    @Bean(name = "primaryEntityManager")
    public EntityManager entityManager(EntityManagerFactoryBuilder builder) {
        return primaryEntityManagerFactory(builder).getObject().createEntityManager();
    }
	
	 /**
   自动注入Jpa配置spring.jpa
   **/
    @Resource 
    private JpaProperties jpaProperties;

	
    /**
    优先加载A实体事务管理器工厂(包含了数据源,实体扫描位置,持久化单元,属性)
    **/
    @Primary
    @Bean(name = "primaryEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean primaryEntityManagerFactory(EntityManagerFactoryBuilder builder) {

        Map<String, Object> properties = hibernateProperties.determineHibernateProperties(
                jpaProperties.getProperties(), new HibernateSettings());
        return builder
                .dataSource(primaryDataSource) // 设置数据源
                .packages("cn.xiaoliu.common.bean.po.primary") //设置A库实体的位置
                .persistenceUnit("primaryPersistenceUnit") //设置持久化单元
                .properties(properties) //设置属性
                .build();
    }
	
    /**
    优先加载A事务管理器
    **/
    @Primary
    @Bean(name = "primaryTransactionManager")
    public PlatformTransactionManager primaryTransactionManager(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(primaryEntityManagerFactory(builder).getObject());
    }
    
}
```
**注意 : springboot2.1.x之后整合JPA配置多数据源时**
**HibernateSettings hibernateSettings = new HibernateSettings();**
**jpaProperties.getHibernateProperties(hibernateSettings)报错**
**(springboot2.1.x之后不再包含此方法)**

## 1.4 加载B数据源
同理设置B数据源

```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
        entityManagerFactoryRef = "secondaryEntityManagerFactory",
        transactionManagerRef = "secondaryTransactionManager",
        basePackages = "cn.xiaoliu.web.repository.secondary")
public class SecondaryConfig {

    @Autowired
    @Qualifier("secondaryDataSource")
    private DataSource secondaryDataSource;

    @Autowired
    private HibernateProperties hibernateProperties;

    @Bean(name = "secondaryEntityManager")
    public EntityManager entityManager(EntityManagerFactoryBuilder builder) {
        return secondaryEntityManagerFactory(builder).getObject().createEntityManager();
    }

    @Resource
    private JpaProperties jpaProperties;

    @Bean(name = "secondaryEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean secondaryEntityManagerFactory(EntityManagerFactoryBuilder builder) {
        Map<String, Object> properties = hibernateProperties.determineHibernateProperties(
                jpaProperties.getProperties(), new HibernateSettings());
        return builder
                .dataSource(secondaryDataSource)
                .packages("cn.xiaoliu.common.bean.po.secondary")
                .persistenceUnit("secondaryPersistenceUnit")
                .properties(properties)
                .build();
    }

    @Bean(name = "secondaryTransactionManager")
    PlatformTransactionManager secondaryTransactionManager(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(secondaryEntityManagerFactory(builder).getObject());
    }

}
```
此时的目录结构为
```txt
├─common
│  └─src
│      └─main
│          └─java
│              └─cn
│                  └─xiaoliu
│                          └─common
│                              ├─bean
│                              │  ├─config
│                              │  ├─dto
│                              │  ├─po ------> 实体
│                              │  │  ├─base
│                              │  │  ├─primary -----> A库
│                              │  │  ├─resp
│                              │  │  └─secondary ----> B库
│                              │  └─result
│                              ├─constants
│                              ├─enums
│                              │  ├─mallUrl
│                              │  └─type
│                              └─utils
├─library
└─web
    └─src
        ├─main
        │  ├─java
        │  │  └─cn
        │  │      └─xiaoliu
        │  │              └─web
        │  │                  ├─annotations
        │  │                  │  └─aspect
        │  │                  ├─configuration
        │  │                  ├─constants
        │  │                  ├─controller
        │  │                  ├─domain
        │  │                  │  ├─reqparam
        │  │                  │  └─respparam
        │  │                  ├─exception
        │  │                  ├─filter
        │  │                  ├─repository ----> 仓库
        │  │                  │  ├─common
        │  │                  │  ├─primary -----> A库
        │  │                  │  └─secondary -----> B库
        │  │                  ├─service
        │  │                  │  └─impl
        │  │                  ├─task
        │  │                  │  └─config
        │  │                  └─utils
        │  │                      └─jpaBathOp

```

## 1.5 数据源下的批量优化操作

```java
1 抽出接口
public interface BatchService {

    /**
     * 批量插入
     *
     * @param list 实体类集合
     * @param <T>  表对应的实体类
     */
     <T> void batchInsert(List<T> list, EntityManager entityManager);

    /**
     * 批量更新
     *
     * @param list 实体类集合
     * @param <T>  表对应的实体类
     */
     <T> void batchUpdate(List<T> list, EntityManager entityManager);
}

/**
公共方法
**/
public class CommonBathOpServiceImpl implements BatchService {
    @Override
    public <T> void batchInsert(List<T> list, EntityManager entityManager) {
        if (!ObjectUtils.isEmpty(list)) {
            for (T t : list) {
                entityManager.persist(t);
            }
            entityManager.flush();
            entityManager.clear();
        }
    }

    @Override
    public <T> void batchUpdate(List<T> list, EntityManager entityManager) {
        if (!ObjectUtils.isEmpty(list)) {
            for (T t : list) {
                entityManager.merge(t);
            }
            entityManager.flush();
            entityManager.clear();
        }
    }
}

/**
A库操作设置
**/
@Service
@Transactional(value = "primaryTransactionManager")
public class PrimaryBatchServiceImpl extends CommonBathOpServiceImpl {

	/**
	这是关键指定A库持久化单元
	**/
    @PersistenceContext(name = "primaryEntityManager",unitName = "primaryPersistenceUnit")
    private EntityManager entityManager;
    /**
     * 批量插入
     *
     * @param list 实体类集合
     * @param <T>  表对应的实体类
     */
    public <T> void batchInsert(List<T> list) {
        super.batchUpdate(list, this.entityManager);
    }

    /**
     * 批量更新
     *
     * @param list 实体类集合
     * @param <T>  表对应的实体类
     */
    public <T> void batchUpdate(List<T> list) {
        super.batchUpdate(list, this.entityManager);
    }
}

/**
B库操作设置
**/
@Service
@Transactional(value = "secondaryTransactionManager")
public class SecondaryBatchServiceImpl extends CommonBathOpServiceImpl {
	
	/**
	这是关键指定B库持久化单元
	**/
    @PersistenceContext(name = "secondaryEntityManager",unitName="secondaryPersistenceUnit")
    private EntityManager entityManager;

    /**
     * 批量插入
     *
     * @param list 实体类集合
     * @param <T>  表对应的实体类
     */
    public <T> void batchInsert(List<T> list) {
        super.batchUpdate(list, this.entityManager);
    }

    /**
     * 批量更新
     *
     * @param list 实体类集合
     * @param <T>  表对应的实体类
     */

    public <T> void batchUpdate(List<T> list) {
        super.batchUpdate(list, this.entityManager);
    }
}
```

# 1.6 A库操作中B库异步操作

在A库中 , 假设我们需要进行B库操作 , 这边业务是进行一个迁移业务 , 对此使用的是异步分页迁移 , 而这个过程就会出现如何保证B库事务的一直性 , 解决方案是加入一个countDownLatch来阻塞线程,在此期间如果出现异常的话 ,则进行回滚

```java
    public Integer g2HServiceLogic(Date startTime, Date endTime, AtomicInteger count, Integer saveType, String tip) throws Exception {
        // 是否存在异常
        AtomicReference<Boolean> isError = new AtomicReference<>(false);
        double pageSize = 10000L;
        log.info("{}开始导出数据", tip);
        count.set(g2HRepository.getGiftIdsPageCount(startTime, endTime));
        if (count.get() == 0) {
            log.info("{}{}>>>>>{}:没有数据", tip, startTime, endTime);
            return 0;
        }
        //获取做大的页数
        long maxPageNum = Double.valueOf(Math.ceil(((double) count.get() / pageSize))).longValue();
        log.info("{}{}>>>>>{},拥有数据条数：{},分页数：{}", tip, startTime, endTime, count.get(), maxPageNum);
        //设置子线程倒计时
        CountDownLatch threadLatch = new CountDownLatch(Long.valueOf(maxPageNum).intValue());
         //设置主线程倒计时
        CountDownLatch mainLatch = new CountDownLatch(Long.valueOf(maxPageNum).intValue());
        long pageSizeLong = BigDecimal.valueOf(pageSize).longValue();
        List<Future<Integer>> taskList = new ArrayList<>();
        Set<String> deleteSet = Collections.synchronizedSet(new HashSet<>());
        //每次分页,独自创建B库自己的事务
        for (long i = 0; i < maxPageNum; i++) {
            long pageNum = i;
            taskList.add(ThreadUtil.executorService.submit(() -> {
                        DefaultTransactionDefinition def = new DefaultTransactionDefinition();
                        def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
                        TransactionStatus status = secondaryTransactionManager.getTransaction(def);
                        Set<String> ids = new HashSet();
                        try {
                            log.info("{}limit: {} >>> pageSize: {} >>> allPage: {}", tip, pageNum, pageSizeLong, maxPageNum);
                            //业务库ids
                            ids = g2HRepository.getGiftIdsPage(startTime, endTime, pageNum * pageSizeLong, pageSizeLong);
                            // 历史库ids
                            List<String> historyIds = sg2HRepository.queryByIds(ids);
                            if (ObjectUtil.isNotEmpty(historyIds)) {
                                log.info("{}>>>有重复,删除历史表量{}", tip, historyIds.size());
                                sg2HRepository.deleteByIds(historyIds);
                            }
                            deleteSet.addAll(ids);
                            log.info("{}>>>>迁移数据量{}", tip, ids.size());
                            if (ObjectUtil.isNotEmpty(ids)) {
                                List<GiftRecEntity> saveList = g2HRepository.queryByIds(ids);
                                List<SGiftRecEntity> t = saveList.stream().map(x -> {
                                    SGiftRecEntity sgift = new SGiftRecEntity();
                                    BeanUtils.copyProperties(x, sgift);
                                    return sgift;
                                }).collect(Collectors.toList());
                                SecondaryBatchServiceImpl.batchUpdate(t);
                            }
                        } catch (Exception e) {
                            // 接受异常 处理异常
                            isError.set(true);
                            e.printStackTrace();
                        } finally {
                            threadLatch.countDown();
                        }
                        try {
                            // 等所有处理结果
                            log.info("{}子线程等待提交", Thread.currentThread().getName());
                            threadLatch.await();
                            int size = ids.size();
                            ids.clear();
                            log.info("{}子线程开始提交", Thread.currentThread().getName());
                            if (isError.get()) {
                                // 事务回滚
                                secondaryTransactionManager.rollback(status);
                                return null;
                            } else {
                                //事务提交
                                secondaryTransactionManager.commit(status);
                                return size;
                            }
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            return null;
                        } finally {
                            mainLatch.countDown();
                        }
                    })
            );
        }

        log.info("{}主线程等待执行", tip);
        mainLatch.await();
        int number = 0;
        int synNumber = 0;
        for (Future<Integer> future : taskList) {
            Integer synCount = future.get();
            if (ObjectUtil.isNotNull(synCount)) {
                number++;
                synNumber += synCount;
            }
        }
        log.info("{}主线程开始执行,数据迁移结束,子线程成功数量{}>>>期望{}", tip, number, maxPageNum);
        log.info("{}子线程同步数量{}>>>期望{}", tip, synNumber, count.get());
        if (synNumber != count.get()) {
            throw new RuntimeException(tip + "同步异常,同步数量" + synNumber + ">>>期望" + count.get());
        }

        if (number == maxPageNum) {
            log.info("{}执行业务库删除,数量为{}", tip, deleteSet.size());
            g2HRepository.deleteByIds(deleteSet);
            return synNumber;
        } else {
            throw new RuntimeException("B库插入有异常,A库任务执行失败");
        }
    }

```









