---
title: MyBaits技术
date: 2020-01-13 17:55:18
tags: [MyBaits,Mysql]
categories: [Java]
---

MyBaits(之前为ibaits)基于 java 的==持久层框架==，它内部封装了 jdbc，使开发者只需要关注 sql 语句本身，而不需要花费精力去处理加载驱动、创建连接、创建 statement 等繁杂的过程。

​	mybatis 通过xml 或注解的方式将要执行的各种statement 配置起来，并通过java 对象和statement 中sql的动态参数进行映射生成最终执行的 sql 语句，最后由 mybatis 框架执行 sql并将结果映射为 java 对象并返回

<!--more-->

## 框架

框架是软件(系统)的半成品，框架封装了很多的细节，使开发者可以使用简单的方式实现功能,大大提高开发效率。 

![k5K4Yv.png](https://t1.picb.cc/uploads/2020/01/11/k5K4Yv.png)

#### 核心配置文件顺序(重要)

配置步骤

1. 利用模版工具导入mybat导入提示

   ```java
   <?xml version="1.0" encoding="UTF-8" ?>
   <!DOCTYPE configuration
           PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
           "http://mybatis.org/dtd/mybatis-3-config.dtd">
   ```

2. 引入外部properties文件,提高复用性

   ```java
   <properties resource="jdbc.properties"></properties>
   ```

3. 引入包扫描配置文件,减少sql语句标签的导入< typeAliases>

   ```java
   <typeAliases>
           <typeAlias type="bean.User" alias="user"></typeAlias>
   </typeAliases>
   ```

4. 配置数据库环境< environments><环境 , 事务 , 连接池>

   ```java
    <!--配置连接数据库的环境 default:指定使用哪一个环境-->
       <environments default="development">
       //默认使用第一个环境,id按第一次配置(随意)
           <environment id="development">
               <!--配置事务,MyBatis事务用的是jdbc-->
               <transactionManager type="JDBC"/>
               <!--配置连接池, POOLED:使用连接池(mybatis内置的); UNPOOLED:不使用连接池-->
               <dataSource type="POOLED">
                   <property name="driver" value="${jdbc.driver}"/>
                   <property name="url" value="${jdbc.url}"/>
                   <property name="username" value="${jdbc.username}"/>
                   <property name="password" value="${jdbc.password}"/>
               </dataSource>
           </environment>
       </environments>
   ```

5. 使用包加载映射配置文件 < mappers>


```java
   <mappers>
        <!--引入xml映射文件; resource属性: 映射文件的路径-->
          //方便批量导入包内所有的配置文件
        <mapper resource="Dao/UserDao.xml"/>
    </mappers>
```

![k50nS8.png](https://t1.picb.cc/uploads/2020/01/11/k50nS8.png)

**properties（引入外部properties文件）**

settings（全局配置参数）

**typeAliases（类型别名）**

typeHandlers（类型处理器）

objectFactory（对象工厂）

plugins（插件）

#### mybaits核心配置文件------properties

为了不写死jdbc的驱动配置

- jdbc.properties

```properties
jdbc.driver=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/mybatis_day01?characterEncoding=utf-8
jdbc.user=root
jdbc.password=123456
```

- 引入到核心配置文件

```xml
<configuration>
   <properties resource="jdbc.properties">
    </properties>
    <!--数据源配置-->
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="UNPOOLED">
                <property name="driver" value="${jdbc.driver}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.user}"/>
                <property name="password" value="${jdbc.password}"/>
            </dataSource>
        </environment>
    </environments>
    ....
</configuration>
```
#### mybaits核心配置文件-----typeAliases（类型别名）

##### 定义单个别名

- 核心配置文件

```xml
<typeAliases>
      <typeAlias type="com.itheima.bean.User" alias="user"></typeAlias>
 </typeAliases>
```

- 修改UserDao.xml

```java
<select id="findAll" resultType="user">
    SELECT  * FROM  user
</select>
```
##### 批量定义别名

使用package定义的别名：就是pojo的类名，大小写都可以

- 核心配置文件

```xml
<typeAliases>
    <package name="com.itheima.bean"/>
</typeAliases>
```

- 修改UserDao.xml

```java
<select id="findAll" resultType="user">
         SELECT  * FROM  user
</select>
```
#### mybaits核心配置文件-----Mapper

进行包扫描加载映射配置文件,批量将包里面的xml导入

##### 方式一:引入映射文件路径

```xml
<mappers>
     <mapper resource="com/itheima/dao/UserDao.xml"/>
 </mappers>
```

##### 方式二:扫描接口

注: 此方式只能用作:代理开发模式,原始dao方式不能使用.

- 配置单个接口

```xml
<mappers>
 	<mapper class="com.itheima.dao.UserDao"></mapper>
</mappers>
```

- 批量配置

  扫描的是编译后文件夹的xml,同目录

```xml
<mappers>
   <package name="com.itheima.dao"></package>
</mappers>
```




工具类中的sqlSession不能声明为成员变量,因为外面应用时有多个SQLSession



#### 自定义映射规则

sql的字段名称与pojo的字段名称不一致时,直接配置,会导致字段名称不匹配进而导致结果为null值

![k50RTL.png](https://t1.picb.cc/uploads/2020/01/11/k50RTL.png)



#### 使用xml配置文件增删改查 

![3cshVK.png](https://s2.ax1x.com/2020/03/01/3cshVK.png)

![3csBUU.png](https://s2.ax1x.com/2020/03/01/3csBUU.png)



#### #和$区别

1. `#{}`表示一个占位符号
   - 通过`#{}`可以实现 preparedStatement 向占位符中设置值,自动进行 java 类型和 数据库 类型转换 
   - `#{}`可以有效防止 sql 注入 
   - `#{}`可以接收简单类型值或 pojo 属性值
   - 如果 parameterType 传输单个简单类型值(String,基本类型)， `#{}` 括号中可以是 value 或其它名称。
2. `${}`表示拼接 sql 串
   - 通过`${}`可以将 parameterType 传入的内容拼接在 sql 中且不进行 jdbc 类型转换. 
   - `${}`不能防止 sql 注入
   - `${}`可以接收简单类型值或 pojo 属性值
   - 如果 parameterType 传输单个简单类型值.`${}`括号中只能是 value 

## Mysql标签属性-----parameterType

1. 传递简单类型

```
#{任意字段}或者${value}
```

2. 传递pojo对象类型

```
#{javaBean属性名}或者${javaBean属性名}
```

3. 传递的包装的pojo

```
#{属性名.属性名} 或者${属性名.属性名} 
```

## Mysql标签属性-----resultType

1. 输出简单类型  直接写 `java类型名`  eg: resultType="int"
2. 输出pojo对象  直接写 `pojo类型名`  eg: User
3. 输出pojo列表类型  写 `列表里面的泛型的类型` eg: List<User>    写User
4. ResultMap
   - 解决查询出来的结果的列名和javaBean属性不一致的请求



## 日志



## Mybatis 连接池与事务

### 数据源

mybaits有三种数据源

unpooled 不使用连接池

 pooled 使用自带连接池



### 事务

源码的opeansseion的实现类default...,中有个一个没有传参自动autocommit为false , 可以在传参时加入true,后面的事务就会自动上交事务



## Mybatis 映射文件的 SQL 

### 动态 SQL 之if标签

if适合动态多条件查询

```java
<if test="user!=null and user.uid=0">
  AND uid = #{user.uid}
</if>
//if的test里面不要加#{} , 用and不是&
```

为了判断传入参数的数量以及是否为""或为null

### 动态 SQL 之where标签

为了简化上面 where 1=1 的条件拼装，我们可以采用< where>标签来简化开发。

< where />可以==自动处理第一个 and , 建议全部加上and==

where标签可以在条件的最前面添加where关键字,并将第一个条件前的and关键字去掉

### 动态标签之foreach标签

foreach标签用于遍历集合，它的属性：

- collection:代表要遍历的集合元素，注意编写时不要写#{}
- open:代表语句的开始部分(一直到动态的值之前)
- close:代表语句结束部分
- item:代表遍历集合的每个元素，生成的变量名(随便取)
- sperator:代表分隔符 (动态值之间的分割)



## sql片段

xml映射的文件中有多数的重复代码,就可以使用sql片段,或者后期需要修改,

sql标签可以把公共的sql语句进行抽取, 再使用include标签引入. 好处:好维护, 提高效率

```java
<sql id="sql" >
       select uid,username,sex,birthday,address from t_user
    </sql>

    <select id="findByIds" parameterType="int" resultType="User">
        <include refid="sql"/>
            <foreach collection="list" item="id"  separator="," open="where uid in (" close=")">
                #{id}
            </foreach>
    </select>
```



# Mybaits缓存

可以减少数据库查询次数,提升系统性能(数据不经常改变,如省份)

​	一级缓存：它是**sqlSession**对象的缓存，自带的(不需要配置)不可卸载的(不想使用还不行).  一级缓存的生命周期与sqlSession一致。(自带)

eg: 相同的sqlsession下,打印两次sql语句,只出现一次sql信息

![k8Ajjw.png](https://t1.picb.cc/uploads/2020/01/12/k8Ajjw.png)

- 一级缓存会在sqlsession对象调用commit\clearCache()\close()方法的时候清除,也会执行增删改后清除

  ​二级缓存：它是**SqlSessionFactory**的缓存。只要是同一个SqlSessionFactory创建的SqlSession就解压共享二级缓存的内容，并且可以操作二级缓存。二级缓存如果要使用的话，需要我们自己手动开启(需要配置的)。

![k8Ab4W.png](https://t1.picb.cc/uploads/2020/01/12/k8Ab4W.png)

注意: 以上两个缓存只有使用了增删改,那么就会清除缓存;当我们在使用二级缓存时，缓存的类一定要实现 java.io.Serializable 接口，这种就可以使用序列化方式来保存对象。



## 多表查询

一对一查询

![k8AaJL.png](https://t1.picb.cc/uploads/2020/01/12/k8AaJL.png)

方式一 : 使用继承的方式来进行两者的关联,但是逻辑上有问题

方式二 : 在 Account 类中加入 User类的对象作为 Account 类的一个属性

需要考虑对应关系,以哪张表为主体进行查询



例如: 账户和人 , 一个账户只能对应一个人的信息 , 所以只能账户封装人的属性

一对多查询

![3csKDP.png](https://s2.ax1x.com/2020/03/01/3csKDP.png)

在一个pojo中添加另外一个pojo的list集合

多对多查询

多对多关系其实我们看成是双向的一对多关系。用户与角色的关系模型就是典型的多对多关系。本质就是多个一对多查询

**注意** : (重要)

一对一 : association标签 的类型是javatype=" "

一对多 : collection 标签的类型是oftype=" "



## 延迟加载

实际开发过程中很多时候我们并不需要总是在加载一方信息时就一定要加载另外一方的信息。 此时就是我们所说的延迟加载。等用到另外一方(和当前关联的那一份)的数据的时候, 再去查询;如果用不到, 不查询.

![3csZ3d.png](https://s2.ax1x.com/2020/03/01/3csZ3d.png)



## 使用注解进行增删改查

增

```java
  @SelectKey(keyColumn = "uid",keyProperty ="uid",before = false,resultType = Integer.class, statement ="select last_insert_id()")
    @Insert("insert into t_user (username,sex,birthday,address) values (#{username},#{sex},#{birthday},#{address})")
    int addUser(User user);
```

删

```java
 @Delete("delete from t_user where uid = #{id}")
    int deleteByUid(Integer uid);
```

改

```java
    @Update("update t_user SET username=#{username},sex=#{sex},birthday=#{birthday},address=#{address} where uid = #{uid}")
    int update(User user);
```

查

```java
 @Select("select * from t_user where uid=#{uid}")
    User findByUid(Integer Uid);
```

模糊查询

```java
  /*@Select("select * from t_user where username like '%${value}%'")*/
    @Select("select * from t_user where username like \"%\"#{str}\"%\" ")
    List<User> searchUser(String str);
```

## 使用注解进行自定义映射

```java
  @Results(value = {
            @Result(column = "uid",property = "userId",id = true),
            @Result(column = "username",property = "userName"),
            @Result(column = "sex",property = "userSex"),
            @Result(column = "birthday",property = "userBirthday"),
            @Result(column = "address",property = "userAddress"),
    })
    @Select("select * from t_user")
    List<anotherUser> findUsers();
```

## 使用注解及延迟加载进行一对一查询

```java
public interface AccountDao  {
        @Results(value = {
                @Result(column = "aid" ,property = "aid",id = true),
                @Result(column = "uid",property = "uid"),
                @Result(column = "uid",property = "user",one = @One(select = "Dao.UserDao.findUser",fetchType = FetchType.LAZY),javaType= User.class )
        })
        @Select("select * from t_account where  aid=#{aid} ")
        Account findByAid(int aid);
  
public interface UserDao {
    @Select("select * from t_user where uid =#{uid}")
    User findUser();
}
```

## 使用注解进行一对多查询

```java
public interface UserDao {
    @Results(value={
            @Result(column = "uid",property = "uid",id=true),
            @Result(column = "uid",property = "accounts" ,many=@Many(select = "Dao.AccountDao.findAccounts",fetchType = FetchType.LAZY))
    })
    @Select("select * from t_user where  uid=#{uid} ")
    User findByUid(Integer uid);
}
public interface AccountDao  {
        @Select("select * from t_account where aid=#{aid}")
        List<Account> findAccounts();
}
```

## 使用注解进行多对多查询

```java
//角度role-->user
public interface RoleDao {
    @Results(value = {
            @Result(column = "rid",property ="rid",id = true),
            @Result(column = "rid",property = "users",many = @Many(select = "Dao.UserDao.findUsers",fetchType = FetchType.LAZY))
    })
    @Select("select * from t_role where rid=#{rid}")
    Role findRole(int rid);
}
public interface UserDao {
    @Select("select * from t_user u,user_role ur,t_role r where u.uid=ur.uid and r.rid=ur.rid and r.rid=#{rid}")
    User findUsers();
}
```

```java
//角度user-->role
public interface UserDao {
    @Results(value = {
            @Result(column = "uid",property = "uid",id = true),
            @Result(column ="uid",property = "roles",many = @Many(select = "Dao.RoleDao.findRoles",fetchType = FetchType.LAZY)),
    })
    @Select("select * from t_user  where uid=#{uid}")
    User findUser(int uid);
}
public interface RoleDao {
    @Select("select * from t_user u,user_role ur,t_role r where u.uid=ur.uid and r.rid=ur.rid and u.uid=#{uid}")
    Role findRoles();
}
```