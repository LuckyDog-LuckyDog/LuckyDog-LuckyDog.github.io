---
title: 基于自制SpringMvc框架的权限校验
date: 2020-02-11 15:54:06
tags: [Mysql ,Tomcat ,invoke ]
categories: [Java]

---

利用之前的自制的Spring框架 ,对用户登录时的权限进行校验 , 达到过滤用户效果, SpringMvc 和Maven环境配置下面文章就不赘述 

<!--more-->

# 效果展示

## 用户登录界面

### 有权限

admin是超级管理员

![kqmB9T.jpg](https://t1.picb.cc/uploads/2020/02/13/kqmB9T.jpg)

zhangsan不是超级管理员

![kqmRAt.jpg](https://t1.picb.cc/uploads/2020/02/13/kqmRAt.jpg)



### 无权限

![kqg6St.png](https://t1.picb.cc/uploads/2020/02/11/kqg6St.png)

![kqgIoF.png](https://t1.picb.cc/uploads/2020/02/11/kqgIoF.png)

# 权限校验思路

```java
1. 登录的时候进行认证(记得存入session中)----(判断是用户名和密码,然后将数据库该用户信息涉及的权限全部存储在容器中,供访问时校验)
2. 设置xml文件,对访问页面需要的路径进行添加,并配置权限,供解析
3. 设置注解 , 对页面请求(.do)进行权限添加 , 供解析
4. 过滤器Filter来捕捉访问页面同时解析xml和注解拥有的权限 然后与session中的用户信息进行对比,即可进行过滤
```



# 代码实现

## 1. 用户认证信息获取

```java
/**
 * author:小刘
 * 日期: 2020/1/15 16:30
 */
public class UserService {
    public User findUser(User loginUser) throws Exception {
            SqlSession sqlSession = SqlSessionFactoryUtils.openSqlSession();
            UserDao userDao = sqlSession.getMapper(UserDao.class);
            User user = userDao.findUserByName(loginUser.getUsername());
            //1.如果有该用户,就检查密码是否正确
            if(user!=null){
                if(user.getPassword().equals(loginUser.getPassword())){
                    List<String> authorityList = new ArrayList<String>();
                    //2. 根据用户获取用户访问页面权限
                    RoleDao roleDao = sqlSession.getMapper(RoleDao.class);
                    List<Role> roleList = roleDao.selectRolesByUserId(user.getId());
                    if(roleList!=null && roleList.size()>0){
                        for (Role role : roleList) {
                            authorityList.add(role.getKeyword());
                            //2.根据角色获得用户操作权限
                            PermissionDao permissionDao = sqlSession.getMapper(PermissionDao.class);
                            List<Permission> permissionList =  permissionDao.selectPermissionsByRoleId(role.getId());
                            if(permissionList!=null && permissionList.size()>0){
                                for (Permission permission : permissionList) {
                                    authorityList.add(permission.getKeyword());
                                }
                            }
                        }
                    }
                    user.setAuthorityList(authorityList);
                    SqlSessionFactoryUtils.commitAndClose(sqlSession);
                    return user;
                }else {
                    throw  new RuntimeException("密码错误 , 请重新输入");}
            }else {
                throw  new RuntimeException("没有该账号,请注册");
            }
        }
```



##2. 访问页面权限Xml配置(要根据数据库定义来哦 ! ) 

```xml
<?xml version="1.0" encoding="utf-8" ?>
<beans>
    <!--授权列表-->
    <security pattern="/pages/index.html" has_role="ROLE_ADMIN,ROLE_QUESTION_RECORDER"/>
    <security pattern="/pages/questionBasicList.html" has_role="ROLE_ADMIN,ROLE_QUESTION_RECORDER"/>
    <security pattern="/pages/questionClassicList.html" has_role="ROLE_ADMIN"/>
    <security pattern="/pages/userList.html" has_role="ROLE_ADMIN"/>
    <!--注解扫描包,是为了扫描时能减少非目标类的扫描,提高效率-->
    <scan package="Controller" />
</beans>
```



## 3. 设置注解类和访问请求上添加注解

```java
//注解类
/**
 * author:小刘
 * 日期: 2020/2/8 10:05
 */
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface PreAuthorize  {
    String value();
}
//访问请求
     /**
     * 添加学科信息的请求
     * @param request
     * @param response
     * @throws IOException
     */
    @PreAuthorize("COURSE_ADD")
    @RequestMapping("/course/addCourse")
    public void addCourse(HttpServletRequest request, HttpServletResponse response) throws IOException {
        try {
            Course course = JsonUtils.parseJSON2Object(request, Course.class);
            course.setCreateDate(DateUtils.parseDate2String(new Date()));
            User user = (User) request.getSession().getAttribute(Constants.USERCONTROLLER_USER);
            course.setUserId(user.getId());
            courseService.addCourse(course);
            JsonUtils.printResult(response,new Result(true,"添加成功"));
        } catch (IOException e) {
            e.printStackTrace();
            JsonUtils.printResult(response,new Result(true,"修改失败"));
        }
    }
```



## 4. 过滤器捕捉,解析,对比

```java
/**
 * author:小刘
 * 日期: 2020/1/28 10:29
 */
public class SecurityFilter implements Filter {
    //容器存放权限
    private static Map<String,String> accessPathMap = new HashMap<String,String>();

    /**
     * 项目启动的时候进行配置文件的读取,提高效率
     * @param config
     * @throws ServletException
     */
    @Override
    public void init(FilterConfig config) throws ServletException {
        InputStream is = null;
        try {
            //获得初始化参数
            String configLocation = config.getInitParameter("configLocation");
            is = SecurityFilter.class.getClassLoader().getResourceAsStream(configLocation);
            //根据xml获得document
            SAXReader saxReader = new SAXReader();
            Document document = saxReader.read(is);
            //获得所有的security标签
            List<Element> securityEles = document.selectNodes("//security");
            //遍历
            if(securityEles!=null && securityEles.size()>0){
                for (Element securityEle : securityEles) {
                    String pattern = securityEle.attributeValue("pattern");
                    String has_role = securityEle.attributeValue("has_role");
                    //存到Map里面
                    accessPathMap.put(pattern,has_role);
                }
            }
            //获取sacn标签
            Element scanEle = (Element) document.selectSingleNode("//scan");
            //获取包里面的相关类
            String scanEleName = scanEle.attributeValue("package");
            List<Class<?>> classList = ClassScannerUtils.getClasssFromPackage(scanEleName);
            if(classList.size()>0 && classList !=null){
                for (Class<?> clazz : classList) {
                    //增加目标类注解判断,减少非目标类的判断
                    Controller controller = clazz.getAnnotation(Controller.class);
                    if(controller!=null){
                        Method[] methods = clazz.getMethods();
                        if(methods!=null&&methods.length>0){
                            for (Method method : methods) {
                                RequestMapping requestMapping = method.getAnnotation(RequestMapping.class);
                                if(requestMapping!=null){
                                    String mappingPath = requestMapping.value();
                                    //将路径做为键 , 将方法及权限信息写入到javabean对象里面
                                    PreAuthorize preAuthorize = method.getAnnotation(PreAuthorize.class);
                                    if(preAuthorize!=null){
                                        String value = preAuthorize.value();
                                        accessPathMap.put(mappingPath,value);
                                    }
                                }
                            }
                        }
                    }
                }
            }
        } catch (DocumentException e) {
            e.printStackTrace();
        }finally {
            //关闭流资源
            if(is!=null){
                try {
                    is.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    /**
     *页面权限过滤
     * @param req
     * @param resp
     * @param chain
     * @throws IOException
     * @throws ServletException
     */
    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) resp;
        //获得当前用户访问的路径
        String contextPath = request.getContextPath();//项目名称
        String requestURI = request.getRequestURI();//项目/请求.do
        String accessPath = requestURI.substring(contextPath.length());
        if(accessPath.endsWith(".do")){  //如果是以.do结尾去掉.do
            accessPath = accessPath.replace(".do","");
        }

        String hasPrivilege = accessPathMap.get(accessPath);
        //权限容器里面是否有需要访问页面或操作的权限
        if(hasPrivilege==null){
            System.out.println("这个路径不受权限控制,放行...");
            chain.doFilter(request,response);
            return;
        }
      
           //判断用户是否登录
        User user = (User) request.getSession().getAttribute(Constants.USERCONTROLLER_USER);
        if(user == null){
            response.sendRedirect(request.getContextPath()+"/login.html");
            return;
        }
        //用户登录后,获取用户的所有权限,判断用户是否有访问页面的权限
        List<String> authorityList = user.getAuthorityList();
        if(authorityList==null || authorityList.size()==0){
            response.getWriter().print("<font color=\"red\" size=\"7\" face=\"微软雅黑\">当前用户没有权限进入，请小劉管理员！</font>");
            return;
        }
        //如果是页面权限就会有一串","数据 , 操作权限就没有
        String[] roleArray = hasPrivilege.split(",");
        //用户是否具备权限
        boolean isAuth =false;
        for (String role : roleArray) {
            if(authorityList.contains(role)){
              //兄弟只要有访问权限,就true
                isAuth = true;
                break;
            }
        }
      
      //兄弟true就放行
        if (isAuth){
            chain.doFilter(request,response);
        }else{
            if (accessPath.endsWith("html")) {
                response.getWriter().print("当前用户权限不足，请切换用户！");
            }else {
                JsonUtils.printResult(response,new Result(flag,"当前用户操作权限不足!"));
            }
        }
    }
```

# 总结

通过Filter进行权限校验时需要注意 , 先要对访问页面的权限与容器权限进行比较放行 , 然后在进行用户是否登录判断, 否则 , 否则就会出现无限执行response.sendRedirect(request.getContextPath()+"/login.html");, 导致浏览器页面出现多次重定向的错误  . 













