---
title: 自定义SpringMvc基本框架
date: 2020-01-14 21:33:47
toc: true
tags: 
- Tomcat
- Servlet
- annotation 
- invoke 
categories: 
- Java
---


通过注解 , 反射 , xml配置来接收特定的请求

<!--more-->

# 以模块为单位创建Servlet

传统方式的开发一个请求对应一个Servlet:这样的话会导致一个模块的Servlet过多,导致整个项目的Servlet都会很多.能不能做一个处理?让一个模块都用一个Servlet处理请求. 用户模块, 创建UserServlet	

注册:http://localhost:8080/day36/userServlet?method=regist
登录:http://localhost:8080/day36/userServlet?method=login
激活:http://localhost:8080/day36/userServle?method=active

但是这样会存在着一个Servlet有多个方法,想要修改就会很难,而且程序很庞大,不好维护

## 优化一
在doGet方法里面,有大量的if语句,可以通过反射来进行优化获得方法调用
```java
class UserServlet extend HttpServlet{
    //找到对应的方法 执行
  	... doGet(HttpServletRequest request, HttpServletResponse response){
      	//1. 获得method请求参数的值【说白了就是方法名】
      	String methodStr =  request.getParameter("method");
        //2.获得字节码对象
        Class clazz  = this.getClass();
        //3.根据方法名反射获得Method
      Method method =   clazz.getMethod(methodStr,HttpServletRequest.class,HttpServletResponse.class);        
        //4.调用
        method.invoke(this,request,response)      
  	}
```

## 优化二
通过进行模块化拆分,每一个模块对应一个Servlet,发现doGet()方法里面,代码都是重复的,所以抽取一个通用的BaseServlet基类, 让各个模块Servlet继承BaseServlet.通用的BaseServlet

```java
@WebServlet("/base")
public class BaseServlet extends HttpServlet {
    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        try {
            //1.获得请求参数method【方法名】  eg:login
            String methodStr = request.getParameter("method");
            //2.获得字节码对象
            Class clazz = this.getClass();
            //3.根据方法名 获得Method
            Method method = clazz.getMethod(methodStr, HttpServletRequest.class, HttpServletResponse.class);
            //4.调用
            method.invoke(this, request, response);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## 优化三
BaseServlet中，可以灵活的处理客户端的请求，应对部分项目开发没有问题。这种实现要求客户端的每个请求都必须传递一个method参数，否则无法找到对应的web方法，另外这个控制器也必须要继承父类BaseServlet，因为是在父类的service()中完成请求解析与调用。这种方式不是很便捷并且耦合度比较高。
   为了让BaseServlet用起来更方便更便捷，让它们的耦合关系更松散，比如写一个总控制器类，这个总控制器继承HttpServlet类，其他的控制器（子控制器）是普通Java类，不需要间接继承HttpServlet，然后由总控制器动态的去调用子控制器的业务方法，这种动态调用是采用在参数中不加入method的方式来实现的

```java
//思路:
//1.获得请求的URI和项目部署路径, 截取获得映路径 eg: /user/login
//2.扫描某个包里面的所有类的字节码对象集合List
//3.遍历字节码对象集合List
//4.获得类里面的所有的Method
//5.遍历所有的Method
//6.获得method上面的RequestMapping注解 获得注解的value属性值
//7.判断value属性值是否和获得映路径一致, 一致 就调用method
```
因为优化三与优化四有些重复 , 转至优化四

## 优化四
优化三存在着如下问题:

1. controller包里面任何一个类都会扫描(获得类里面的Method), 加载 --> 扫描太多
2. 等来了请求再去解析,再去反射创建对象调用 ---> 影响处理请求的速度的
3. 扫描的包名写死了

思路

1. 服务器启动的时候,扫描controller包下面的所有的类 获得字节码对象集合
2. 遍历字节码对象集合, 获得method对象数组
3. 遍历method对象数组, 获得method上面的RequetMapping注解的MappingPath
4. 把MappingPath和method关联存到容器里面
5. 在DispacherServlet的service()方法里面

### web配置及项目结构
![kAV2l6.png](https://t1.picb.cc/uploads/2020/01/14/kAV2l6.png)

### 扫描包工具类
```java
public class ClassScannerUtils {
    /**
     * 获得包下面的所有的class
     * @param
     * @return List包含所有class的实例
     */
    public static List<Class<?>> getClasssFromPackage(String packageName) {
        List clazzs = new ArrayList<>();
        // 是否循环搜索子包
        boolean recursive = true;
        // 包名对应的路径名称并转换
        String packageDirName = packageName.replace('.', '/');
        Enumeration<URL> dirs;
        try {
            dirs = Thread.currentThread().getContextClassLoader().getResources(packageDirName);//file://.......
            while (dirs.hasMoreElements()) {
                URL url = dirs.nextElement();
                String protocol = url.getProtocol();
                if ("file".equals(protocol)) {
                    String filePath = URLDecoder.decode(url.getFile(), "UTF-8");
                    findClassInPackageByFile(packageName, filePath, recursive, clazzs);
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return clazzs;
    }

    /**
     * 在package对应的路径下找到所有的class
     */
    public static void findClassInPackageByFile(String packageName, String filePath, final boolean recursive,
                                                List<Class<?>> clazzs) {
        File dir = new File(filePath);
        if (!dir.exists() || !dir.isDirectory()) {
            return;
        }
        // 在给定的目录下找到所有的文件，并且进行条件过滤
        File[] dirFiles = dir.listFiles(new FileFilter() {

            public boolean accept(File file) {
                boolean acceptDir = recursive && file.isDirectory();// 接受dir目录
                boolean acceptClass = file.getName().endsWith("class");// 接受class文件
                return acceptDir || acceptClass;
            }
        });

        for (File file : dirFiles) {
            if (file.isDirectory()) {
                findClassInPackageByFile(packageName + "." + file.getName(), file.getAbsolutePath(), recursive, clazzs);
            } else {
                String className = file.getName().substring(0, file.getName().length() - 6);//Controller.class
                try {
                    clazzs.add(Thread.currentThread().getContextClassLoader().loadClass(packageName + "." + className));
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

### 注解类

```java
//这个类是为了注明方法上的特定路径
@Target({ElementType.METHOD,ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface RequestMapping {
    String value();
}

//这个类是为了减少非目标类的扫描
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Controller {
}

```

### javaBean类
```java
public class MvcMethod implements Serializable {
    private Method method;//存反射类里面的方法
    private Object obj;//存反射对象
    private Class clazz;//存反射类

    public MvcMethod() {
    }
    public MvcMethod(Method method, Object obj, Class clazz) {
        this.method = method;
        this.obj = obj;
        this.clazz = clazz;
    }

    public Method getMethod() {
        return method;
    }

    public void setMethod(Method method) {
        this.method = method;
    }

    public Object getObj() {
        return obj;
    }

    public void setObj(Object obj) {
        this.obj = obj;
    }

    public Class getClazz() {
        return clazz;
    }

    public void setClazz(Class clazz) {
        this.clazz = clazz;
    }
}
```

### Servlet类
```java
package xiaoliu.main;

import xiaoliu.Utils.ClassScannerUtils;

import javax.servlet.ServletConfig;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * author:小刘
 * 日期: 2020/1/13 20:29
 */
public class DispatcherServlet extends HttpServlet {
    private static Map<String, MvcMethod> methodMap = new HashMap<>();

    @Override
    public void init(ServletConfig config) throws ServletException {
        super.init(config);
        //初始化就调用,降少读取次数 , 已在配置文件配置
        try {
            String aPackaging = config.getInitParameter("Packaging");
            List<Class<?>> classList = ClassScannerUtils.getClasssFromPackage(aPackaging);
            if(classList!=null&&classList.size()>0){
                for (Class<?> clazz : classList) {
                    //增加目标类注解判断,减少非目标类的判断
                    if(clazz.isAnnotationPresent(Controller.class)){
                        Method[] methods = clazz.getMethods();
                        if(methods!=null&&methods.length>0){
                            for (Method method : methods) {
                                RequestMapping requestMapping = method.getAnnotation(RequestMapping.class);
                                if(requestMapping!=null){
                                    String mappingPath = requestMapping.value();
                                    //将路径做为键 , 将方法及对象信息写入到javabean对象里面
                                    methodMap.put(mappingPath,new MvcMethod(method,clazz.newInstance(),clazz));
                                }
                            }
                        }
                    }


                }
            }
        } catch (Exception e) {

        }
    }

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        try {
            //1.获得请求路径
            String requestURI = req.getRequestURI();
            //2.获得项目部署目录
            String contextPath = req.getContextPath();
            //3.截取字符串获得映射路径
            String mappingPath = requestURI.substring(contextPath.length(), requestURI.lastIndexOf("."));
            //4.从Map里面取出Method
            MvcMethod mvcMethod = methodMap.get(mappingPath);
            if(methodMap.containsKey(mappingPath)){//避免拿不到方法
                //5.调用
                MvcMethod mvcMethod = methodMap.get(mappingPath);
                Method method = mvcMethod.getMethod();
                method.invoke(mvcMethod.getObj(),req,resp);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### 控制类

```java
package xiaoliu.main;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
/**
 * author:小刘
 * 日期: 2020/1/13 20:11
 */
@Controller
public class UserController {
    @RequestMapping("/user/login")
    public  void login(HttpServletRequest request, HttpServletResponse response) throws IOException {
        System.out.println("login...");
        response.getWriter().print("login...");
    }
    @RequestMapping("/user/register")
    public void regist(HttpServletRequest request,HttpServletResponse response) throws IOException {
        System.out.println("register...");
        response.getWriter().print("register....");
    }
}
```























