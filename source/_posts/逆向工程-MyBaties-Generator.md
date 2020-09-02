---
title: 逆向工程-MyBaties Generator
date: 2020-09-02 11:55:27
tags:
- mybaits
- 逆向工程
categories:
- java
---

之前项目是hibernate通过实体创建表 , 记录MyBatis Generator通过sql语句创建表之后 , 逆向生成实体及方法
<!--more-->

# MyBatis Generator

## 快速入门
**参考官方文档快速入门**: http://mybatis.org/generator/generatedobjects/exampleClassUsage.html
下面的配置 , 官方文档的参数都是有的

### XML配置
**参考资料**:https://blog.csdn.net/qigc_0529/article/details/80704330?utm_medium=distribute.pc_feed_404.none-task-blog-BlogCommendFromBaidu-3.nonecase&depth_1-utm_source=distribute.pc_feed_404.none-task-blog-BlogCommendFromBaidu-3.nonecas
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
  PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
  "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>

  <context id="DB2Tables" targetRuntime="MyBatis3">
  
      <!-- 逆向生成时，没有注释 -->
    <!-- <commentGenerator>
      <property name="suppressAllComments" value="true" />
    </commentGenerator> -->
    
     <commentGenerator type="com.eyed.app.util.MybatisCommentGenerator">
            <!--<property name="suppressDate" value="true"/>-->
            <!-- <property name="suppressAllComments" value="true" /> -->
            <!-- 是否生成注释代时间戳-->
            <property name="suppressDate" value="false"/>
        </commentGenerator>
    
      <!-- 配置数据库连接信息 -->
    <jdbcConnection 
        driverClass="com.mysql.jdbc.Driver"
        connectionURL="jdbc:mysql://192.168.2.177:3306/eyed2?testonborrow=true&amp;useUnicode=true&amp;autoReconnect=true&amp;characterEncoding=utf-8&amp;rewriteBatchedStatements=TRUE&amp;allowMultiQueries=true&amp;useSSL=false"
        userId="root"
        password="123456">
    </jdbcConnection>
    
    <!-- java类型解析 -->
    <javaTypeResolver >
      <property name="forceBigDecimals" value="false" />
    </javaTypeResolver>
    
    <!-- 指定javaBean生成的位置
        targetPackage：生成在哪个包下
        targetProject：生成在哪个工程下
     -->
    <javaModelGenerator 
        targetPackage="com.eyed.app.bean.mapping" 
        targetProject=".\src\main\java">
      <property name="enableSubPackages" value="true" />
      <property name="trimStrings" value="true" />
    </javaModelGenerator>
    
    <!-- 指定sql映射文件生成的位置 -->
    <sqlMapGenerator 
        targetPackage="mapper"  
        targetProject=".\src\main\resources\com\eyed\app">
      <property name="enableSubPackages" value="true" />
    </sqlMapGenerator>

    <!-- 指定dao接口生成的位置,就是mapper接口生成的位置 -->
    <javaClientGenerator type="XMLMAPPER" 
        targetPackage="com.eyed.app.dao"  
        targetProject=".\src\main\java"
        >
      <property name="enableSubPackages" value="true" />
    </javaClientGenerator>

    <!-- table 指定每个表的生成策略 
        tableName 表示连向数据后逆向生成哪张表
        domainObjectName 表示表对应的Javabean的类名
    -->
    <table tableName="activity_road_trip" domainObjectName="ActivityRoadTrip" mapperName="ActivityRoadTripDao"></table>
    <table tableName="activity_road_trip_order" domainObjectName="ActivityRoadTripOrder" mapperName="ActivityRoadTripOrderDao"></table>
    <table tableName="activity_road_trip_user" domainObjectName="ActivityRoadTripUser" mapperName="ActivityRoadTripUserDao"></table>
    
  </context>
</generatorConfiguration>
```
### 文档注释修改
**public class DefaultCommentGenerator implements CommentGenerator这个类的提示对我们来说很不友好 , 所以需要自定义**
```java
public class MybatisCommentGenerator implements CommentGenerator {

    private Properties properties;
    private Properties systemPro;
    private boolean suppressDate;
    private boolean suppressAllComments;
    private String nowTime;

    public MybatisCommentGenerator() {
        super();
        properties = new Properties();
        systemPro = System.getProperties();
        suppressDate = false;
        suppressAllComments = false;
        nowTime = (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss")).format(new Date());
    }


    /**
     * 类的注释
     * @param innerClass
     * @param introspectedTable
     * @param markAsDoNotDelete
     */
    @Override
    public void addClassComment(InnerClass innerClass, IntrospectedTable introspectedTable, boolean markAsDoNotDelete) {
//        if (suppressAllComments) {
//            return;
//        }
        StringBuilder sb = new StringBuilder();
        innerClass.addJavaDocLine("/**");
        sb.append(" * ");
        sb.append(introspectedTable.getFullyQualifiedTable());
        innerClass.addJavaDocLine(sb.toString().replace("\n", " "));
        sb.setLength(0);
        sb.append(" * @comment: ");
        sb.append(" * @author ");
        sb.append(systemPro.getProperty("user.name"));
        sb.append(" * @date ");
        sb.append(nowTime);
        innerClass.addJavaDocLine(" */");
    }

    @Override
    public void addClassComment(InnerClass innerClass, IntrospectedTable introspectedTable) {
        if (suppressAllComments) {
            return;
        }
        StringBuilder sb = new StringBuilder();
        innerClass.addJavaDocLine("/**");
        sb.append(" * ");
        sb.append(introspectedTable.getFullyQualifiedTable());
        sb.append(" ");
        sb.append(getDateString());
        innerClass.addJavaDocLine(sb.toString().replace("\n", " "));
        innerClass.addJavaDocLine(" */");
    }

    /**
     * 设置字段注释
     */
    @Override
    public void addFieldComment(Field field, IntrospectedTable introspectedTable, IntrospectedColumn introspectedColumn) {
        if (suppressAllComments) {
            return;
        }
        StringBuilder sb = new StringBuilder();
        field.addJavaDocLine("/**");
        sb.append(" * ");
        sb.append(introspectedColumn.getRemarks() + " " + introspectedColumn.getActualColumnName());
        field.addJavaDocLine(sb.toString().replace("\n", " "));
        field.addJavaDocLine(" */");
    }

    @Override
    public void addFieldComment(Field field, IntrospectedTable introspectedTable) {
        if (suppressAllComments) {
            return;
        }
        StringBuilder sb = new StringBuilder();
        field.addJavaDocLine("/**");
        sb.append(" * ");
        sb.append(introspectedTable.getFullyQualifiedTable());
        field.addJavaDocLine(sb.toString().replace("\n", " "));
        field.addJavaDocLine(" */");
    }

    /**
     * 设置setter方法注释
     */
    @Override
    public void addSetterComment(Method method, IntrospectedTable introspectedTable,
                                 IntrospectedColumn introspectedColumn) {
        if (suppressAllComments) {
            return;
        }
        method.addJavaDocLine("/**");
        StringBuilder sb = new StringBuilder();
        sb.append(" * ");
        sb.append(introspectedColumn.getRemarks());
        method.addJavaDocLine(sb.toString().replace("\n", " "));
        sb.setLength(0);
//        sb.append(" * @author ");
//        sb.append(systemPro.getProperty("user.name"));
//        method.addJavaDocLine(sb.toString().replace("\n", " "));
        sb.setLength(0);
        if(suppressDate){
            sb.append(" * @date " + nowTime);
            method.addJavaDocLine(sb.toString().replace("\n", " "));
            sb.setLength(0);
        }
        Parameter parm = method.getParameters().get(0);
        sb.append(" * @param ");
        sb.append(parm.getName());
        sb.append(" ");
        sb.append(introspectedColumn.getRemarks());
        method.addJavaDocLine(sb.toString().replace("\n", " "));
        method.addJavaDocLine(" */");
    }

    /**
     * 设置getter方法注释
     */
    @Override
    public void addGetterComment(Method method, IntrospectedTable introspectedTable,
                                 IntrospectedColumn introspectedColumn) {
        if (suppressAllComments) {
            return;
        }
        method.addJavaDocLine("/**");
        StringBuilder sb = new StringBuilder();
        sb.append(" * ");
        sb.append(introspectedColumn.getRemarks());
        method.addJavaDocLine(sb.toString().replace("\n", " "));
        sb.setLength(0);
//        sb.append(" * @author ");
//        sb.append(systemPro.getProperty("user.name"));
        method.addJavaDocLine(sb.toString().replace("\n", " "));
        sb.setLength(0);
        if(suppressDate){
            sb.append(" * @date " + nowTime);
            method.addJavaDocLine(sb.toString().replace("\n", " "));
            sb.setLength(0);
        }
        sb.append(" * @return ");
        sb.append(introspectedColumn.getActualColumnName());
        sb.append(" ");
        sb.append(introspectedColumn.getRemarks());
        method.addJavaDocLine(sb.toString().replace("\n", " "));
        method.addJavaDocLine(" */");
    }

    @Override
    public void addJavaFileComment(CompilationUnit compilationUnit) {
        if (suppressAllComments) {
            return;
        }
        return;
    }

    @Override
    public void addComment(XmlElement xmlElement) {
        return;
    }

    @Override
    public void addRootComment(XmlElement rootElement) {
        return;
    }

    @Override
    public void addConfigurationProperties(Properties properties) {
        this.properties.putAll(properties);
        suppressDate = Boolean.valueOf(properties.getProperty(PropertyRegistry.COMMENT_GENERATOR_SUPPRESS_DATE));
        suppressAllComments = Boolean.valueOf(properties.getProperty(PropertyRegistry.COMMENT_GENERATOR_SUPPRESS_ALL_COMMENTS));
    }

    protected void addJavadocTag(JavaElement javaElement, boolean markAsDoNotDelete) {
        javaElement.addJavaDocLine(" *");
        StringBuilder sb = new StringBuilder();
        sb.append(" * ");
        sb.append(MergeConstants.NEW_ELEMENT_TAG);
        if (markAsDoNotDelete) {
            sb.append(" do_not_delete_during_merge");
        }
        String s = getDateString();
        if (s != null) {
            sb.append(' ');
            sb.append(s);
        }
        javaElement.addJavaDocLine(sb.toString());
    }

    protected String getDateString() {
        String result = null;
        if (!suppressDate) {
            result = nowTime;
        }
        return result;
    }
```
### 生成逆向文件

```java
public class MBGGenerator {
	private static Logger log = LoggerFactory.getLogger(MBGGenerator.class);
	
	public static void main(String[] args) throws Exception {
        List<String> warnings = new ArrayList<String>();
          boolean overwrite = true;
          String path = MBGGenerator.class.getClassLoader().getResource("").getFile();
          log.info("[项目绝对路径]{}",path);
          File configFile = new File(path+"mbg.xml");
          ConfigurationParser cp = new ConfigurationParser(warnings);
          Configuration config = cp.parseConfiguration(configFile);
          DefaultShellCallback callback = new DefaultShellCallback(overwrite);
          MyBatisGenerator myBatisGenerator = new MyBatisGenerator(config, callback, warnings);
          myBatisGenerator.generate(null);
   }
}
```
