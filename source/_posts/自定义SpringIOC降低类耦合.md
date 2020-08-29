---
title: 自定义SpringIOC降低类耦合
date: 2020-02-13 15:20:49
tags: [多态 , xml, invoke ]
categories: [Java]
---

通过一步步实现自定义SpringIOC , 降低类的耦合性 

<!--more-->

# 前言

早期 JDBC 操作，注册驱动时，我们为什么不使用 DriverManager 的 register 方法，而是采
用 Class.forName 的方式？

​	原因就是： 我们的类依赖了数据库的具体驱动类（MySQL） ，如果这时候更换了数据库（比如 Oracle） ，
需要修改源码来重新数据库驱动.

## 解决上述耦合的思路

类与类之间的耦合的根本原因是:在一个类中new了另外一个类的对象

思路 : 不在本类中创建另外一个类的对象, 由Spring 或者其它帮忙创建类

​	通过反射来注册驱动的，代码如下：

```java 
Class.forName("com.mysql.jdbc.Driver");//此处只是一个字符串
```

​	此时的好处是，我们的类中不再依赖具体的驱动类，此时就算删除 mysql 的驱动 jar 包，依然可以编译（运行无法通过,因为缺少类啊） 。 

​	但是又存在了一个新的问题， mysql 驱动的全限定类名''字符串''是在 java 类中写死的(硬编码)，当要修改的时候要修改源码。可以使用配置文件配置(XML)解决

##  自定义解耦版本1

```java
public class UserController {
    public static void main(String[] args) {
        UserController userController = new UserController();
        userController.getName();
    }
      public void getName(){
       System.out.println("小劉");
    }
}
```

上述程序如果出现了异常就会导致整个程序的奔溃,因为耦合性很强

思路 : 使用接口(多态)来进行解耦

```java
//接口
public interface UserService {
    String getName();
}
//接口实现类
public class UserServiceImpl implements UserService{
    @Override
    public String getName() {
        return "小劉";
    }
}
//主类
public class UserController {
    public static void main(String[] args) {
        UserService userService = new UserServiceImpl();
        userService.getName();
    }
}
```

## 自定义解耦版本2

上述代码在主程序中new一个新的对象 , 导致new的类与主程序类形成了强耦合

思路 : 在别的类中new新的对象------新建一个实现工厂

```java
//接口
public interface UserService {
    String getName();
}
//接口实现类
public class UserServiceImpl implements UserService{
    @Override
    public String getName() {
        return "小劉";
    }
}
//工厂类
public class BeanFactory {
    public static Object getBean(String className){
        try {
          //创建新类 UserService userService = new UserServiceImpl();为了能复用使用反射
            return Class.forName(className).newInstance();
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException("反射创建对象失败");
        }
    }
}
//主类
public class UserController {
    public static void main(String[] args) {
        BeanFactory beanFactory = new BeanFactory("xiaoliu.service.UserServiceImpl");
        UserService userService=(UserServiceImpl)beanFactory.getBean();
      	userService.getName();
    }
}
```

## 自定义解耦版本3

上述代码存在硬编码问题 

思路 : 使用xml配置文件来进行解耦

```java
//接口
public interface UserService {
    String getName();
}
//接口实现类
public class UserServiceImpl implements UserService{
    @Override
    public String getName() {
        return "小劉";
    }
}
//工厂类
public class BeanFactory {
    public static Object getBean(String id){
          //1. 解析beans.xml文件，根据id获取要创建对象的类的全限定名
        SAXReader reader = new SAXReader();
        //将beans.xml文件转换成字节输入流
        InputStream is = BeanFactory.class.getClassLoader().getResourceAsStream("beans.xml");
        //2. 读取配置文件，得到document对象
        try {
            Document document = reader.read(is);
            //3. 查找所有的bean标签
            List<Element> list = document.selectNodes("//bean");
            if (list != null && list.size() > 0) {
                //遍历出每一个bean标签
                for (Element element : list) {
                    //获取每一个bean标签的id属性和class属性
                    String attrId = element.attributeValue("id");
                  //如果
                  if(attrId !=null && attrId.equals("id") ){
                    String attrClass = element.attributeValue("class");
                    //根据类的全限定名，创建对象
                    Object obj = Class.forName(attrClass).newInstance();
                        return obj;
                  }else{
                    throw new RuntimeException("配置文件参数错误,请检查");
                  }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
          throw new RuntimeException("读取配置文件创建对象失败");
        }
    }
}
//xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans>
    <bean id="userService" class="xiaoliu.service.UserServiceImpl"/>
</beans>
//主类
public class UserController {
    public void getName(){
       UserService userService = (UserService) BeanFactory.getBean("userService");//创建UserServiceImpl类的对象(与实现类解耦)
       String name = userService.getName();
       System.out.println(name);
    }
    public static void main(String[] args) {
        UserController userController = new UserController();
        userController.getName();
    }
}
```

## 自定义解耦版本3优化

上述代码中 , 主程序每次调用一个getBean方法就会读取一次配置文件,但是在多人协作时可以独立开发,只要工作者实现目标接口 , 配置xml就能实现类的创建,不影响他人开发

思路 : 让读取的配置文件出现在静态代码块 , 减少资源开销 , 同时将创建的对象存入一个容器中 , 需要时取出即可

```java
//这里只贴工厂类 , 其它不变
public class BeanFactory {
  //创建一个容器 ,因为需要指定对象,创建Map ,这个map就是存放id和对象的键值对
    private static Map<String,Object> beanMap = new HashMap<>();
    static {
        //解析beans.xml文件,获取所有的id和class的值，并且根据class的值创建对象，然后将id作为key，对象作为value存储到beanMap中
        SAXReader reader = new SAXReader();
        InputStream is = BeanFactory.class.getClassLoader().getResourceAsStream("beans.xml");
        try {
            Document document = reader.read(is);
            List<Element> list = document.selectNodes("//bean");
            if (list != null && list.size() > 0) {
                for (Element element : list) {
                    String attrId = element.attributeValue("id");
                    String attrClass = element.attributeValue("class");
                    Object obj = Class.forName(attrClass).newInstance();
                    beanMap.put(attrId,obj);
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
           throw new RuntimeException("读取配置文件创建对象失败");
        }
    }
    public static Object getBean(String id){
        //你传id给我，我直接将这个id对应的对象给你
        return beanMap.get(id);
    }
}
```

## 总结

写到最后颇有点SpringIoc创建对象的感觉 , 也就是Spring创建单例对象的底层吧,哈哈!















