---
title: 自定义DBUTILS
date: 2019-12-22 17:23:15
tags: [Mysql]
categories: [Java]
---


DBUTILS的可以操作数据库 , 突发奇想想山寨一个,于是........

<!-- more -->

# 自定义连接池

之前由于对MapListHandler功能的理解有误,该对象并不是通过反射去获取值,而是直接通过sql预编译获得的结果集进行操作

Connection对象在JDBC使用的时候就会去创建一个对象,使用结束以后就会将这个对象给销毁了(close).每次创建和销毁对象都是耗时操作.需要使用连接池对其进行优化.程序初始化的时候，初始化多个连接,将多个连接放入到池(集合)中.每次获取的时候,都可以直接从连接池中进行获取.使用结束以后,将连接归还到池中.
- 使用连接池的目的: 可以让连接得到复用, 避免浪费


下图是自定义的连接池+装饰者+jdbc工具类(很low , 见怪)
jdbc工具类

```java
import java.io.IOException;
import java.sql.*;
import java.util.Properties;

/**
 * author:小刘
 * 日期: 2019/12/17 13:38
 */
public class jdbcUTILS {
    private static Properties properties;
    static String jdbcclassname;
    static String url;
    static String user;
    static String password;
    static Connection conn;

    static {
        try {
            properties = new Properties();
           properties.load(jdbc1.class.getClassLoader().getResourceAsStream("jdbc.properties"));
            jdbcclassname = properties.getProperty("JDBCCLASSNAME");
            url = properties.getProperty("URL");
            user = properties.getProperty("USER");
            password = properties.getProperty("PASSWORD");
            Class.forName(jdbcclassname);
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
    public static Connection getConnection() throws SQLException {
        return    DriverManager.getConnection(jdbcUTILS.url, jdbcUTILS.user, jdbcUTILS.password);
    }
    public static void close(Connection co, Statement statement) {
        close(null, co, statement);
    }
    public static void close(ResultSet re, Connection co, Statement statement) {
        try {
            if (re != null) {
                re.close();
            }
            if (co != null) {
                co.close();
            }
            statement.close();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

装饰者(增强连接池的close)

```java
package MyDataSource;
import java.sql.*;
import java.util.LinkedList;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.Executor;
/**
 * author:小刘
 * 日期: 2019/12/19 10:29
 */
public class MyConnection implements Connection {
    private LinkedList<Connection> pool;
    private Connection connection; //被增强的对象
    //在MyConnection里面需要得到被增强的connection对象(通过构造方法传进去)
    public MyConnection(LinkedList<Connection> pool, Connection connection) {
        this.pool = pool;
        this.connection = connection;
    }
    @Override
    //改写close()的逻辑, 变成归还
    public void close() throws SQLException {
        pool.addLast(connection);
    }
    @Override
    //其它方法的逻辑, 还是调用被增强connection对象之前的逻辑
    public PreparedStatement prepareStatement(String sql) throws SQLException {
        return connection.prepareStatement(sql);
    }
    @Override
    public Statement createStatement() throws SQLException {
        return connection.createStatement();
    }
/.......后面还有好多重写省略
}
```

连接池--新建的连接调用close会被销毁

 ```java
package MyDataSource;
import javax.lang.model.element.VariableElement;
import javax.sql.DataSource;
import java.io.DataOutputStream;
import java.io.PrintWriter;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;
import java.sql.SQLFeatureNotSupportedException;
import java.util.ArrayList;
import java.util.LinkedList;
import java.util.logging.Logger;
/**
 * author:小刘
 * 日期: 2019/12/18 10:04
 */
public class MyDataSource implements DataSource {
    private static LinkedList<Connection> pool;
    //程序初始化的时候, 创建5个连接 存到LinkedList
    static {
        try {
            pool = new LinkedList();
            for (int i = 0; i < 5; i++) {
                Connection connection = jdbcUTILS.getConnection();
                MyConnection myConnection = new MyConnection(pool, connection);
                pool.add(myConnection);//连接池里面预设的连接数量
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
     // 返回池子里面连接的数量
    public  int getCount(){
        return pool.size();
    }
    @Override
    public Connection getConnection() throws SQLException {
        System.out.println(pool.size());
        //连接池没有就创建
        if(pool.size()==0){
            Connection connection1 = DriverManager.getConnection("jdbc:mysql://localhost/mysqltest2","root","root");
            return connection1;
        }else{
        Connection connection = pool.removeFirst();  //被增强的connection
        return connection;}
    }
 ```



由于笔者比较菜,许多因素未考虑到,俗话说工欲善其事,必先利其器,为此选择连接池和JDBC工具包来提高效率

# C3P0 --创建连接池

C3P0是一个开源的JDBC连接池，它实现了数据源和JNDI绑定，支持JDBC3规范和JDBC2的标准扩展。C3P0是异步操作的，所以一些操作时间过长的JDBC通过其它的辅助线程完成。目前使用它的开源项目有Hibernate，Spring等。C3P0有自动回收空闲连接功能
这里我使用的是IDEA创建-----C3P0 包来完成对连接池对象的创建和sql数据库连接对象的调用方法

```java
package C3P0;
import com.mchange.v2.c3p0.ComboPooledDataSource;
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
/**
 * author:小刘
 * 日期: 2019/12/18 20:27
 */
public class C3P0Utils {
//    private static final DataSource dataSource = new ComboPooledDataSource();
    private static DataSource dataSource;
    public  static DataSource getDataSource(){
        //自动读取配置文件
        return dataSource = new ComboPooledDataSource();
    }
    public static Connection getConnection() throws Exception {
        Connection connection = dataSource.getConnection();
        return connection;
    }
    public static void closeAll(ResultSet resultSet, Statement statement, Connection connection) throws SQLException {
        if (resultSet != null) {
            resultSet.close();
        }
        if (statement != null) {
            statement.close();
        }
        if (connection != null) {
            connection.close();
        }
    }
}
```




**这里有几点需要注意 :**

- jar里面的配置文件必须拷贝到src项目下的resource标识文件夹中
- 配置的文件不能更改
- 配置的文件的属性名不能修改,否则框架读取不到信息
  [![ku0rbe.png](https://t1.picb.cc/uploads/2019/12/21/ku0rbe.png)](https://www.picb.cc/image/ku0rbe)
  [![ku0Mh6.png](https://t1.picb.cc/uploads/2019/12/21/ku0Mh6.png)](https://www.picb.cc/image/ku0Mh6)

# DBUTILS
DbUtils是Apache组织提供的一个对JDBC进行简单封装的开源工具类库，使用它能够简化JDBC应用程序的开发，同时也不会影响程序的性能 ,简而言之就是-----简化JDBC操作数据库的步骤

## JavaBean

数据库查询出来的数据一定会有地方来进行存储 , 封装数据 , 它就是javabean(创建一个Person类) , 只是多了如下要求 : 

- 私有字段  
- 提供公共的get/set方法
- 无参构造
- 实现Serializable
- 字段:  全局/成员变量  eg: `private String username`
- 属性: 去掉get或者set首字母变小写  eg: `setUsername-去掉set->Username-首字母变小写->username`


## 利用DBUTILS进行增删改

![kuSlY0.png](https://t1.picb.cc/uploads/2019/12/21/kuSlY0.png)

## 利用DBUTILS进行查,结果封装在JavaBean里面

![kuSCu1.png](https://t1.picb.cc/uploads/2019/12/21/kuSCu1.png)


# 自定义DBUTILS功能----增删改查

秉承着知其然 , 亦知其所以然 , 自己玩玩o(∩_∩)o 哈哈

## 增删改
```java
package DBUtils;
import C3P0.C3P0Utils;
import org.apache.commons.dbutils.ResultSetHandler;
import org.apache.commons.dbutils.handlers.BeanHandler;
import org.apache.commons.dbutils.handlers.BeanListHandler;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.sql.*;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
/**
 * author:小刘
 * 日期: 2019/12/19 18:27
 */
public class SelfDefineDbUtils {
    private static Connection connection;
    //类加载就创建连接对象,减少内存开销
    static {
        try {
            connection = C3P0Utils.getDataSource().getConnection();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    public static int Update(String sql, Object... parameter) {
        try {
            PreparedStatement pstm = connection.prepareStatement(sql);
            int count = pstm.getParameterMetaData().getParameterCount();
            if (count == parameter.length) {
                for (int i = 0; i < count; i++) {
                    pstm.setObject(i + 1, parameter[i]);
                }
                return pstm.executeUpdate();
            } else {
                throw new RuntimeException("paraments does not be match !");
            }
        } catch (SQLException e) {
            e.printStackTrace();
            throw new RuntimeException("unpredicat error");
        }
    }
```

## 查
分析BeanHandler , MapHandler , BeanListHandler , MapListHandler发现他们一个共同的巴巴--ResultSetHandler , 这里我只对BeanHandler进行分析 , 其中Type就是beanhandler传入的字节码对象 , 根据底层源码描述
```java
  /**
     * Create a JavaBean from the column values in one <code>ResultSet</code>
     * row.  The <code>ResultSet</code> should be positioned on a valid row before
     * passing it to this method.  Implementations of this method must not
     * alter the row position of the <code>ResultSet</code>.
     * @param <T> The type of bean to create 创建bean对象的类型
     * @param rs ResultSet that supplies the bean data 结果集rs提供数据
     * @param type Class from which to create the bean instance 对象的字节码
     * @throws SQLException if a database access error occurs
     * @return the newly created bean 返回一个新的bean
     */
       <T> T toBean(ResultSet rs, Class<T> type) throws SQLException;
```
![kuS46u.png](https://t1.picb.cc/uploads/2019/12/21/kuS46u.png)

一句话 : bean做为一个载体封装sql查询的数据 , ResultSetHandler做为获取bean对象和sql结果集的中间商
依照这个思路,这里我**重载**单条查询和多条查询的query方法 ,开工 ,见下图

- **单条查询**

  ```java
   public static <T> T query(String sql, ResultSetHandler<T> handler, Object... parament) throws Exception {
          // 获取传入的handler的成员变量
          Field[] fields = handler.getClass().getDeclaredFields();
          Method[] methods = handler.getClass().getDeclaredMethods();
          //获取 type 字段的字节码对象
          fields[0].setAccessible(true);
          Object o = fields[0].get(handler);
          //获取泛型T的对象
          Class o2 = (Class) o;
          T o1 = (T) o2.getConstructor().newInstance();
          //获取sql预编译集合
          PreparedStatement pstm = connection.prepareStatement(sql);
       //判断传参
          if (sql == null || handler == null) {
              throw new RuntimeException("Miss one of parament of sql or handler !");
          }
          //这里还需要判断parament是否是存在
          //存在
          if (parament.length != 0) {
              for (int i = 1; i < parament.length + 1; i++) {
                  pstm.setObject(i, parament[i - 1]);
                  getDataAndSetData((Class) o, o1, pstm );
              }
          } else {
              //不存在
              //结果集
              getDataAndSetData((Class) o, o1, pstm);
          }
              return o1;
      }
  ```

*getDataAandSetDate方法*

```java
public static <T> void getDataAndSetData(Class o, T o1, PreparedStatement pstm) throws Exception{
         ArrayList<T> list;
        HashMap<String, T> map;
        ResultSet resultSet = pstm.executeQuery();
        //元属性
        ResultSetMetaData metaData = pstm.getMetaData();
        int count = metaData.getColumnCount();
        resultSet.next();
            //获得类对象的成员属性数组
            Class<?> o1Class = o1.getClass();
            for (int i = 0; i < count; i++) {
                //获取数据库字段名
                String name = metaData.getColumnName(i + 1);
                //获取数据库每列字段结果
                Object data = resultSet.getObject(name);
                //方法集
                Method[] mt = o1Class.getDeclaredMethods();
                for (int j = 0; j < mt.length; j++) {
                    if (mt[j].getName().equalsIgnoreCase("set" + name)) {
                        Field field = o.getDeclaredField(name);
                        field.setAccessible(true);
                        field.set(o1, data);
                    }
                }
            }
        }
```



*单条查询测试结果*
![kuBYFD.png](https://t1.picb.cc/uploads/2019/12/21/kuBYFD.png)

- **多条查询**
  *list*

  ```java
  public static <T> List<T> query(String sql, ResultSetHandler<T> handler, Object... parament) throws Exception {
          // 获取传入的handler的成员变量
          Field[] fields = handler.getClass().getDeclaredFields();
          Method[] methods = handler.getClass().getDeclaredMethods();
          //获取 type 字段的字节码对象
          fields[0].setAccessible(true);
          Object o = fields[0].get(handler);
          //获取泛型T的对象
          Class o2 = (Class) o;
          T o1 = (T) o2.getConstructor().newInstance();
          //获取sql预编译集合
          PreparedStatement pstm = connection.prepareStatement(sql);
         //判断传参
          if (sql == null || handler == null) {
              throw new RuntimeException("Miss one of parament of sql or handler !");
          }
          //这里还需要判断parament是否是存在
          //存在
          if (parament.length != 0) {
              for (int i = 1; i < parament.length + 1; i++) {
                  pstm.setObject(i, parament[i - 1]);
                 return getDataAndSetListData((Class) o, o1, pstm,fields );
              }
          } else {
              //不存在
              //结果集
             return getDataAndSetListData((Class) o, o1, pstm,fields);
          }
          return null;
      }
  ```

  ​

*getDataAandListSetDate方法*

```java
 public static <T> List<T> getDataAndSetListData(Class o, T o1, PreparedStatement pstm,Field[] fields) throws Exception {
         ArrayList<T> list = new ArrayList<>();
        ResultSet resultSet = pstm.executeQuery();
        //元属性
        ResultSetMetaData metaData = pstm.getMetaData();
        int count = metaData.getColumnCount();
        while (resultSet.next()) {
            //获得类对象的成员属性数组
            Class<?> o1Class = o1.getClass();
            T o3 = (T)o1Class.newInstance();
            for (int i = 0; i < count; i++) {
                //获取数据库字段名
                String name = metaData.getColumnName(i + 1);
                //获取数据库每列字段结果
                Object data = resultSet.getObject(name);
                //方法集
                Method[] mt = o1Class.getDeclaredMethods();
                for (int j = 0; j < mt.length; j++) {
                    if (mt[j].getName().equalsIgnoreCase("set" + name)) {
                        Field field = o.getDeclaredField(name);
                        field.setAccessible(true);
                        field.set(o3, data);
                    }

                }
            }
            list.add(o3);
        }
        return list;
    }
```

*测试结果*
![kuFepc.png](https://t1.picb.cc/uploads/2019/12/21/kuFepc.png)

*Map*--------这里的思想是将查询到的数据库数据直接封装在Map,然后在封装在List

```java
    public static <T> List<Map<String,Object>> query(String sql,ResultSetHandler<T> handler, Object... parament) throws Exception {
        PreparedStatement pstm = connection.prepareStatement(sql);
        //这里还需要判断parament是否是存在
        //存在
        if (sql == null || handler == null) {
            throw new RuntimeException("Miss one of parament of sql or handler !");
        }
        if (parament.length != 0) {
            for (int i = 1; i < parament.length + 1; i++) {
                pstm.setObject(i, parament[i - 1]);
                return getDataAndSetMapData(pstm);
            }
        } else {
            //不存在
            //结果集
           return getDataAndSetMapData(pstm);
        }
        return null;
    }
```

getDataAndSetMapData方法

```java

public static  List<Map<String,Object>> getDataAndSetMapData(PreparedStatement pstm) throws Exception{
        List<Map<String,Object>> list = new ArrayList<>();
        ResultSet resultSet = pstm.executeQuery();
        //元属性
        ResultSetMetaData metaData = pstm.getMetaData();
        int columnCount = metaData.getColumnCount();
        while (resultSet.next()) {
            Map<String,Object> map = new HashMap<>();
            for (int i=1;i<=columnCount;i++){
                String columnName = metaData.getColumnName(i);
                Object value = resultSet.getObject(columnName);
                map.put(columnName,value);
            }
            list.add(map);
        }
        return list;
    }
```

测试结果

![kuLNRR.png](https://t1.picb.cc/uploads/2019/12/22/kuLNRR.png)

# 总结 
通过这次 , 加深了对反射和泛型的运用 ;同时书写一个程序时,要先考虑清楚目标,理清思路然后写Bug , 最后再调试Bug----o(∩_∩)o 哈哈

