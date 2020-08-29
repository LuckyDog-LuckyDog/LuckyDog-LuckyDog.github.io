---
title: 汤姆猫之Session
date: 2019-12-27 20:29:19
tags: [Tomcat,JSP]
categories: [Java]
---

Tomcat , a lovely cat~ ~

<!--more-->

# 汤姆猫服务器之Session

session是服务器端的技术。服务器为每一个浏览器开辟一块内存空间，即session对象。由于session对象是每一个浏览器特有的，所有用户的记录可以存放在session对象中。同时，每一个session对象都对应一个sessionId，服务器把sessionId写到cookie中，再次访问的时候，浏览器把sessionId带过来，找到对应的session对象



## 为什么有了Cookie还有Session?

cookie只能存放字符串,而session能够存放任意类型的数据Object
cookied存在数据的大小有限制,而session存放数据没有大小限制
cookie存放数据不安全,session存放在服务器安全性高
但是cookie能够实现关闭浏览器之后几天还能存放数据,session没有,而且session的底层是依赖cookie , 所以两者无法互相取代

## Session的执行原理
- Servlet获取Cookie中的信息
  1、获得cookie中传递过来的SessionId(cookie)
  2、如果Cookie中没有Jsessionid,则创建session对象
  3、如果Cookie中有sessionid,找指定的session对象
     如果有sessionid并且session对象存在，则直接使用
     如果有sessionid，但session对象销毁了，则执行第二步

### 关闭浏览器之后,能否获取之前session中存取的数据?

不能 , 因为cookie默认的生命周期是一次会话请求,所以携带cookie的sessionID就会被销毁,下次访问请求,服务器就找不到cookiedID

### 关闭浏览器之后,之前的session对象销毁没有?

没有,session对象存放在服务器上

### 之前的session没有销毁 ,那么为什么获取不到他里面数据?

没有销毁,session是基于cookie, jsessionId保存到cookie里面的, 默认情况下cookie是会话级别,浏览器关闭了cookie就是消失了,也就是说==sessionId消失了==, 从而找不到对应的session对象了, 就不能使用了.
​	解决方案: 自己获得sessionId, 自己写给浏览器 设置Cookie的有效时长, 这个Cookie的key必须: `JSESSIONID`
```java
String id =request.getsession().getId();
Cookie cookie = new Cookie("JSESSIONID",id);
cookie.setMaxAge = (7*24*60*60);
respone.addCookie(cookied);
```
### (题外)三个域对象比较 

| 域对象                | 销毁                                       | 创建                                       | 作用范围     | 应用场景                                   |
| ------------------ | ---------------------------------------- | ---------------------------------------- | -------- | -------------------------------------- |
| ServletContext     | 服务器正常关闭/项目从服务器移除                         | 服务器启动                                    | 整个项目     | 记录访问次数,聊天室                             |
| HttpSession        | session过期（默认30分钟）/调用invalidate(）方法/服务器==异常==关闭 | ==没有sessionID调 用==request.getSession()方法 | 会话(多次请求) | 验证码校验, ==保存用户登录状态==等                   |
| HttpServletRequest | 响应这个请求(或者请求已经接收了)                        | 来了请求                                     | 一次请求     | servletA和jsp（servletB）之间数据传递(转发的时候存数据) |

### 三个域对象怎么选择?

​	一般情况下, 最小的可以解决就用最小的.(eg: 重定向, 多次请求, 会话范围, 用session;  如果是转发,一般选择request)

## JSP

为了展示动态数据,将html与java结合生成的技术
1. JSP: java 服务器页面, 本质是Servlet
2. JSP产生的原因: Servlet在动态展示很麻烦, jsp动态展示方便一点

### JSP脚本
| 类型               | 翻译成Servlet对应的部分                      | 注意          |
| ---------------- | ------------------------------------ | ----------- |
| <%...%>:Java程序片段 | 翻译成Service()方法里面的内容, 局部的             |             |
| <%=...%>:输出表达式   | 翻译成Service()方法里面的内容,相当于调用out.print() | 输出表达式不能以;结尾 |
| <%!...%>:声明成员变量  | 翻译成Servlet类里面的内容                     |             |
### JSP注释

| 注释类型(3种)                |
| ----------------------- |
| HTML注释<!--HTML注释-->     |
| JAVA注释     //; /* */    |
| JSP注释;     <%--注释内容--%> |

# 打码环节

为了加深对Session和JSP的理解和练习 , 做了如下练习,先上图
## 界面展示(有点丑,不,是很丑,哈哈)
![kCXg4c.png](https://t1.picb.cc/uploads/2019/12/27/kCXg4c.png)

## 效果展示
![kCXkbK.png](https://t1.picb.cc/uploads/2019/12/27/kCXkbK.png)

## 打码

- 处理按钮+的代码
```java
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;
import java.io.IOException;
import java.util.*;
/**
 * author:小刘
 * 日期: 2019/12/27 17:07
 */
@WebServlet(value = "/ChooseServlet")
public class ChooseServlet extends HttpServlet {
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        doGet(request, response);
    }

    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        request.setCharacterEncoding("utf-8");
        HttpSession session = request.getSession();
        //应该要连数据库的,时间紧,用数组测试
        String[] productNames ={"xiaomi","apple","jialaoban","jiazhensan","wangxiaoer"};
        //获取jsp传输过来的index
        int index = Integer.parseInt(request.getParameter("name"));
        //获取请求的商品名称
        String productName = productNames[index];
        System.out.println(productName);
        //获取"Car"Session
        LinkedHashMap<String, Integer> map = (LinkedHashMap<String, Integer>)session.getAttribute("Car");
        //如果用户第一次,就没有session,那就没有map,那增加一个容器
        if(map==null){
             map = new LinkedHashMap<String, Integer>();
            map.put(productName,1);
        }else {
            //判断用户是否有添加商品
            Integer number = map.get(productName);
            if(map.get(productName)!=null){
                //有,+1
             map.put(productName,number+1);
            }else {
                //没有,赋值1
                map.put(productName,1);
            }
        }
        System.out.println(map);
        //将信息转发给Car Servlet
        session.setAttribute("Car",map);
        //回转页面
        response.sendRedirect("index.jsp");
    }
}
```

- 处理按钮-的代码
```java
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;
import java.io.IOException;
import java.util.LinkedHashMap;
/**
 * author:小刘
 * 日期: 2019/12/27 20:18
 */
@WebServlet(value = "/Choose")
public class Choose extends HttpServlet {
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        doGet(request, response);
    }
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        request.setCharacterEncoding("utf-8");
        HttpSession session = request.getSession();
        String[] productNames ={"xiaomi","apple","jialaoban","jiazhensan","wangxiaoer"};
        int index = Integer.parseInt(request.getParameter("name"));
        String productName = productNames[index];
        System.out.println(productName);
        LinkedHashMap<String, Integer> map = (LinkedHashMap<String, Integer>)session.getAttribute("Car");
        //如果没有session,增加容器
        //防止第一可能按 - 按钮导致负数
        if(map==null){
            map = new LinkedHashMap<String, Integer>();
            map.put(productName,0);
        }else {
            Integer number = map.get(productName);
            if(map.get(productName)!=null){
            //防止出现负数
                if(number-1>=0){
                    map.put(productName,number-1);
                }else {map.put(productName,0);}
            }else {
                map.put(productName,0);
            }
        }
        System.out.println(map);
        //将信息转发给Car Servlet
        session.setAttribute("Car",map);
        response.sendRedirect("index.jsp");
    }
}
```

- 购物车代码
```java
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;
import java.io.IOException;
import java.util.Map;
import java.util.Set;

/**
 * author:小刘
 * 日期: 2019/12/27 16:57
 */
@WebServlet(value = "/Car")
public class CarServlet extends HttpServlet {
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        doGet(request, response);
    }
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        //回应浏览器按utf-8解码
        response.setContentType("text/html;charset=utf-8");
        HttpSession session = request.getSession();
       Map<String,Integer> map = (Map)session.getAttribute("Car");
            Set<String> strings = map.keySet();
            for (String string : strings) {
                Integer number = map.get(string);
                //将数据显示在浏览器上
                response.getWriter().write("<h1>"+string+"数量为"+number+"</h1><br/>");
            }
    }
}
```

- JSP(html与java结合体)

```java
<html>
  <head>
    <title>$Title$</title>
      <%
          Integer number = 0;
          Integer number1= 0;
          Integer number2= 0;
          Integer number4= 0;
          Integer number3= 0;
          Map<String, Integer> map = (Map) session.getAttribute("Car");
          if(map!=null){
              Set<String> strings = map.keySet();
              for (String string : strings) {
                  if ("xiaomi".equals(string)) {
                      number = map.get("xiaomi");
                  }
                  if ("apple".equals(string)) {
                      number1 = map.get("apple");
                  }
                  if ("jialaoban".equals(string)) {
                      number2 = map.get("jialaoban");
                  }
                  if ("jiazhensan".equals(string)) {
                      number3 = map.get("jiazhensan");
                  }
                  if ("wangxiaoer".equals(string)) {
                      number4 = map.get("wangxiaoer");
                  }
              }
          }
      %>
  </head>
  <center>
  <body>
  <h1>商品列表</h1>
        小米<a href="ChooseServlet?name=0" target="_self" a><input type="button" name="0" value="+" ></a><%=number%><a href="Choose?name=0" target="_self"><input type="button" name="0" value="-" ></a><br>
        苹果<a href="ChooseServlet?name=1"target="_self"><input type="button"  name="1" value="+"></a><%=number1%><a href="Choose?name=1"target="_self"><input type="button"  name="1" value="-"></a><br>
        家老班<a href="ChooseServlet?name=2"target="_self"><input type="button" name="2"value="+" ></a><%=number2%><a href="Choose?name=2"target="_self"><input type="button" name="2"value="-" ></a><br>
        家政三<a href="ChooseServlet?name=3"target="_self"><input type="button"name="3" value="+" ></a><%=number3%><a href="Choose?name=3"target="_self"><input type="button"name="3" value="-" ></a><br>
        王小二<a href="ChooseServlet?name=4"target="_self"><input type="button"name="4" value="+" ></a><%=number4%><a href="Choose?name=4"target="_self"><input type="button"name="4" value="-" ></a><br>
       <a href="Car">跳转至购物车</a> <br>
  </body>
  </center>
</html>
```
# 总结

JSP中内置了session对象 ,查看了D:\Java\jdk1.8.0_171\apache-tomcat-8.5.27\work\Catalina\localhost\ROOT\org\apache\jsp\index_jsp.java的转译文件,发现java已经自动帮我们创建了 , 加入不想自动添加,在开头写上session=false 就行

- 汤姆猫的配置文件web.xml

发现原来tomcat是通过 jspServlet和 defaultServlet来进行处理请求 ,启动服务器自动加载,等级为3
```jsp
<servlet>
        <servlet-name>jsp</servlet-name>
        <servlet-class>org.apache.jasper.servlet.JspServlet</servlet-class>
        <init-param>
            <param-name>fork</param-name>
            <param-value>false</param-value>
        </init-param>
        <init-param>
            <param-name>xpoweredBy</param-name>
            <param-value>false</param-value>
        </init-param>
        <load-on-startup>3</load-on-startup>
</servlet>
   <!-- The mapping for the default servlet -->
<servlet-mapping>
<servlet-name>default</servlet-name>
        <url-pattern>/</url-pattern>
</servlet-mapping>

    <!-- The mappings for the JSP servlet -->
<servlet-mapping>
        <servlet-name>jsp</servlet-name>
        <url-pattern>*.jsp</url-pattern>
        <url-pattern>*.jspx</url-pattern>
</servlet-mapping>
    
```

- 写到这里突然发现还有一个清空购物车的功能未写,即清除session
```java
/**
 * author:小刘
 * 日期: 2019/12/27 21:24
 */
@WebServlet(value = "/ClearServlet")
public class ClearServlet extends HttpServlet {
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        doGet(request, response);
    }
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        HttpSession session = request.getSession();
        if(session!=null){
            //session销毁
            session.invalidate();
        }
        response.sendRedirect("index.jsp");
    }
}
//jsp
  <a href="ClearServlet">清空购物车</a><br>

```



