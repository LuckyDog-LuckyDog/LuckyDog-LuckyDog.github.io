---
title: spring监听器与zookeeper的配置中心应用
date: 2020-03-01 10:43:37
tags: 
- Zookeeper
- dubbo
- spring
categories: 
- Java
---

记录利用spring监听加载配置属性与zookeeper的配置中心结合 , 达到zookper节点信息改变, 服务器内所有的信息都改变 , 提高效率

<!--more-->

# 框架技术简介

Dubbo是一款高性能的Java **RPC框架**。其前身是阿里巴巴公司开源的一个高性能、轻量级的开源Java RPC框架，可以和Spring框架无缝集成

ZooKeeper , 分布式文件存储系统管理多个服务 , 类似于磁盘上目录树 , 主要用于维护和监控存储的数据变化 , 适用于存储和协同相关的数据 , 不适合用于大数据量存储



## 配置中心与spring结合

### 1. 将jdbc配置文件信息,写入到zookeeper的节点上

```java
RetryPolicy retryPolicy = new ExponentialBackoffRetry(3000,1);
CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 3000, 3000, retryPolicy);
client.start();
client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath("/config/jdbc.url","jdbc:mysql://localhost:3306/dubbo?characterEncoding=utf-8".getBytes()); client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath("/config/jdbc.driver","com.mysql.jdbc.Driver".getBytes());
client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath("/config/jdbc.username","root".getBytes());
client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath("/config/jdbc.password","root".getBytes());
client.close();
```

### 2. 获取zookeeper上节点的信息

将获取的节点信息写入到spring中的**processProperties**中的属性集里面

```java
  // 将jdbc配置信息 , 载入zookeeper中
    private void loadZk(Properties props){
        //创建重试策略
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(3000,1);
        //创建客户端
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 3000, 3000, retryPolicy);
        client.start();
        try {
            byte[] urlBytes = client.getData().forPath("/config/jdbc.url");
            props.setProperty("jdbc.url",new String(urlBytes));
            byte[] userBytes = client.getData().forPath("/config/jdbc.username");
            props.setProperty("jdbc.user",new String(userBytes));
            byte[] pwdBytes = client.getData().forPath("/config/jdbc.password");
            props.setProperty("jdbc.password",new String(pwdBytes));
            byte[] driverBytes = client.getData().forPath("/config/jdbc.driver");
            props.setProperty("jdbc.driver",new String(driverBytes));
        } catch (Exception e) {
            e.printStackTrace();
        }
      client.close();
    }
```

### 3.继承spring框架的读取配置文件类

类中**继承PropertyPlaceholderConfigure **, **重写processProperties** , 将属性集pros 提供该加载方法

```java
public class SettingCenterUtil extends PropertyPlaceholderConfigurer {
    @Override
    protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess, Properties props) throws BeansException {
      loadZk(props);
      super.processProperties(beanFactoryToProcess, props);
    }
```

### 4.配置spring-dao.xml

将外部属性输入改为 类载入 , 由spring核心容器进行管理

```xml
<bean id="settingCenterUtil" class="com.itheima.utils.SettingCenterUtil"></bean>
```

### 5.监听zookeeper节点数据的变化(watch机制)

类 实现一个名为**ApplicationContextAware**的接口,然后重写里面的 **setApplicationContext**方法 ,获取一个可以刷新的配置XmlWebApplicationContext 的对象;调用refresh方法进行属性刷新

```java
private void addWatch(Properties props) {
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(3000,1);
    CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 3000, 3000, retryPolicy);
    client.start();
    //创建缓存机制，监听 /config 路径
    TreeCache treeCache = new TreeCache(client,"/config");
    //启动缓存机制
    try {
        treeCache.start();
    } catch (Exception e) {
        e.printStackTrace();
    }
    //添加监听
    treeCache.getListenable().addListener(new TreeCacheListener() {
        /**
         *
         * @param client  zookeeper客户端对象
         * @param event  zookeeper改变触发的事件
         * @throws Exception
         */
        @Override
        public void childEvent(CuratorFramework client, TreeCacheEvent event) throws Exception {
            //如果是修改事件则，更新数据
            if(event.getType()==TreeCacheEvent.Type.NODE_UPDATED){
                //如果修改的是jdbc.url路径，修改jdbc.url的配置
                if(event.getData().getPath().equals("/config/jdbc.url")){
                    props.setProperty("jdbc.url",new String(event.getData().getData()));
                }else if(event.getData().getPath().equals("/config/jdbc.driver")){
                    props.setProperty("jdbc.driver",new String(event.getData().getData()));
                }else if(event.getData().getPath().equals("/config/jdbc.user")){
                    props.setProperty("jdbc.user",new String(event.getData().getData()));
                }else if(event.getData().getPath().equals("/config/jdbc.password")){
                    props.setProperty("jdbc.password",new String(event.getData().getData()));
                }
                //修改完成后必须刷新spring容器的对象
                applicationContext.refresh();
            }
        }
    });
}
    XmlWebApplicationContext applicationContext;
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = (XmlWebApplicationContext) applicationContext;
    }
```

### 6.加载spring监听器

```xml
   <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring*.xml</param-value>
    </context-param>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
```



















